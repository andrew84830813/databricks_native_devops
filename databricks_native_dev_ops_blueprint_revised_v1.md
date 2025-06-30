# Databricks-Native DevOps Blueprint

A fully “Databricks-only” approach—no local environment mirroring, no in-repo wheel packaging—yet enterprise-grade in security, rollback, CI/CD, and consistency. This blueprint provides a detailed, narrative-style walkthrough of each component to help new team members understand not just the steps, but the reasoning and interconnections that make this pipeline robust and maintainable.

---

## 1. Repository Layout – Why We Organize This Way

**What happens:** When a developer opens the repository they see a consistent, opinionated folder structure that separates concerns clearly.\
**Why it’s needed:** Large projects can quickly become chaotic without conventions. By establishing a predictable layout, we reduce friction on day one, making it easy to locate code, understand its purpose, and add new components without confusion.\
**How it fits:** This structure underpins every subsequent process: CI scripts know where to find tests, the init script knows where to install dependencies, and the Asset Bundle tooling can automatically locate notebooks for job creation.

```text
your-project/
├── deep_project_core/        ← Portable business logic: reusable Python modules without Databricks-specific imports
├── notebooks/                ← User-facing artifacts: exploration, production workflows, and smoke tests
│   ├── eda/                  ← Exploratory data analysis (ad-hoc, non-scheduled)
│   ├── workflows/            ← Production notebooks (1:1 mapping to Jobs)
│   └── smoke_tests/          ← Lightweight notebooks that invoke pytest to validate pipeline entry points
├── requirements/             ← Four-stage dependency management (baseline, constraints, additions, final)
├── init/                     ← Cluster initialization scripts to enforce a clean environment
├── tests/                    ← Unit, integration, and data fixtures
├── config/                   ← Environment configuration: JSON templates, parameter files, secrets mappings
├── databricks.yml            ← Deployment manifest describing jobs, clusters, init scripts per environment
└── .github/workflows/ci.yml  ← GitHub Actions pipeline for testing, scanning, and deployments
```

### Additional Explanation

- ``: Organizes pure Python logic so it can be tested locally and reused in other contexts (e.g., a REST API). By avoiding `dbutils` or Spark imports here, we keep this code decoupled from Databricks specifics.
- ``** vs **``: EDA notebooks are for one-off analysis—never scheduled—while `workflows/` holds the production-ready notebooks. This prevents accidental scheduling of experimental code.
- ``: Centralizing job JSON templates and environment variables (e.g., S3 paths, database URIs) in `config/` ensures that changes to parameters are tracked in Git and not scattered across scripts.

---

## 2. Dependency Workflow – Ensuring Reproducible, Evolvable Environments

**What happens:** We capture the exact set of libraries Databricks provides, transform those into constraints, declare our own additions, resolve all versions into a pinned list, and enforce that list at every cluster startup.\
**Why it’s needed:** Databricks ships hundreds of packages by default. Without controlling the versions, a simple `pip install` can silently upgrade or downgrade dependencies, leading to "works on my cluster" issues. This workflow prevents those surprises and lets us confidently add new libraries over time.

### 2.1 Snapshot the Exact DBR Baseline

Run this once whenever we upgrade to a new Databricks Runtime version:

```python
import pkg_resources, datetime, pathlib

entries = sorted(f"{p.project_name}=={p.version}"
                 for p in pkg_resources.working_set)
header = f"# DBR16.4 ML snapshot {datetime.datetime.utcnow().isoformat()} UTC\n\n"
pathlib.Path("requirements/runtime-baseline.in") \
        .write_text(header + "\n".join(entries) + "\n")
```

- **Outcome:** A plaintext list of exactly what Databricks pre-installs.
- **Purpose:** Serves as the immutable snapshot we compare against when adding or updating libraries.

### 2.2 Convert to “≤” Constraints

To allow patch-level updates if necessary, we convert each `pkg==ver` into `pkg<=ver`:

```python
from pathlib import Path

inp = Path("requirements/runtime-baseline.in")
out = Path("requirements/runtime-baseline.constraints")
lines = inp.read_text().splitlines()
with out.open("w") as f:
    for line in lines:
        if line.startswith("#") or "==" not in line:
            f.write(line + "\n")
        else:
            pkg, ver = line.split("==",1)
            f.write(f"{pkg}<={ver}\n")
```

- **Why:** This enforces an upper bound at the Databricks-provided version, while allowing dependencies that fall below that version if required by new packages.

### 2.3 Declare Direct Dependencies

In `requirements/additional-packages.in`, list only the libraries your code directly depends on (e.g. `pandas==2.2.0`). This file represents the minimal set of new requirements beyond the DBR baseline.

### 2.4 Resolve & Pin with pip-compile

```bash
cd requirements/
pip install --quiet pip-tools
pip-compile \
  additional-packages.in \
  --constraint=runtime-baseline.constraints \
  --output-file=requirements-final.txt \
  --allow-unsafe
```

- **What:** Merges your additions with the DBR constraints, resolves all transitive dependencies, and produces a fully pinned `requirements-final.txt`.
- **Why:** Detects version conflicts early (CI will fail) and ensures every package — direct or transitive — is locked to a known version.

### 2.5 Enforcing on Cluster Startup

Every cluster’s init script (`init/setup-env.sh`) uninstalls any stray packages and reinstalls exactly those in `requirements-final.txt`:

```bash
pip freeze | xargs pip uninstall -y
pip install -r requirements-final.txt
```

This guarantees that interactive notebooks and scheduled jobs run with the exact same environment that passed CI.

### 2.6 Live Constraint-Bumping Process

When a new library demands a version above the current DBR ceiling:

1. Install the library interactively in Dev, verify functionality.
2. Add it to `additional-packages.in` and re-run pip-compile.
3. If pip-compile fails, decide to bump the DBR baseline (by generating a new snapshot) or pin the library to a lower version.
4. Run unit and integration tests on Dev.
5. Open a PR including updated files and let CI validate.
6. Merge once all checks pass.

Every change to `runtime-baseline.*`, `additional-packages.in`, or `requirements-final.txt` is reviewed in Git, providing a full audit trail of dependency evolution.

---

## 3. Cluster Roles & Repo Mounts – Siloed Environments in One Workspace

We maintain three distinct file-system mounts within a single Databricks workspace:

| Mount Path       | Git Ref              | Purpose                   |
| ---------------- | -------------------- | ------------------------- |
| `/Repos/...-dev` | `main` / `feature/*` | Developer sandbox & QA    |
| `/Repos/...-stg` | `release/x.y-RC`     | Pre-production validation |
| `/Repos/...`     | `release/x.y`        | Production pipelines      |

- **Asset Bundles**: Use `databricks bundle deploy --target <env> --branch <ref>` to check out the correct branch into each mount without altering job definitions.
- **RBAC**: Assign Databricks workspace and storage permissions per mount so Dev users cannot accidentally write to Prod.
- **Promotion**: Automate branch promotion via a CI step or dedicated CLI, ensuring consistent, auditable deployments.

---

## 4. Testing Pyramid – Multi-Layered Validation

1. **Unit tests (tests/unit/):** Fast, pure-Python checks that catch logic bugs in isolation.
2. **Integration tests (tests/integration/):** Run the full Spark pipeline on small JSON fixtures to validate schema, joins, and transformations.
3. **Smoke tests (notebooks/smoke\_tests/):** Execute the actual notebook entry points via a pytest wrapper to verify cluster, init scripts, and job configurations.

**Why each layer matters:**

- Unit tests give immediate feedback on small code changes.
- Integration tests run in the real runtime, catching environment or data issues.
- Smoke tests confirm that the production Job wrapper will succeed, preventing silent misconfigurations.

---

## 5. Security & Compliance – Automated Scans Everywhere

| Tool           | Purpose                                | Invocation                                                        |
| -------------- | -------------------------------------- | ----------------------------------------------------------------- |
| `pip-audit`    | CVE and license metadata               | `pip-audit --require-licenses -r requirements-final.txt`          |
| `safety`       | Vulnerability checks against PyPI DB   | `safety check --file=requirements-final.txt`                      |
| `pip-licenses` | License classification and enforcement | `pip-licenses --from=requirements-final.txt --fail-on="GPL;LGPL"` |

- **CI Enforcement:** Any flagged issue blocks merges.
- **Weekly Live Scans:** A scheduled Job re-runs these scanners on the running cluster, alerting Slack for newly disclosed vulnerabilities.

---

## 6. Rollback Strategy – Fast, Atomic Recovery

- Maintain two recent release branches (current and previous).

- **Rollback Command:**

  ```bash
  databricks bundle deploy --target prod --branch release/x.y-1
  ```

- **Automated Drills:** Quarterly sandbox exercises that perform rollback followed by forward deployment, validating runbooks and tooling.

- **Canary Releases:** For critical paths, route a small percentage of traffic to the new release, monitor success, then complete the cutover.

**Why:** Minimizes Mean Time To Recovery (MTTR) by providing a single-command rollback and rehearsed procedures.

---

## 7. CI/CD Workflow – Gatekeeping & Observability

```yaml
jobs:
  unit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Set up Python
        uses: actions/setup-python@v2
        with:
          python-version: 3.9
      - name: Run unit tests
        run: pytest tests/unit
      - name: Security scans
        run: |
          pip-audit --require-licenses -r requirements-final.txt
          safety check --file=requirements-final.txt
          pip-licenses --from=requirements-final.txt --fail-on="GPL;LGPL"

  integration:
    needs: unit
    steps:
      - name: Deploy to Dev
        run: databricks bundle deploy --target dev --branch ${{ github.head_ref }}
      - name: Smoke test in Dev
        run: |
          databricks runs submit --job-id ${{ env.DEV_SMOKE_JOB_ID }}
          # Poll until SUCCESS or timeout

  deploy-prod:
    needs: [unit, integration]
    if: startsWith(github.ref, 'refs/heads/release/')
    steps:
      - name: Approval
        uses: hmarr/auto-approve-action@v1
      - name: Deploy to Prod
        run: databricks bundle deploy --target prod --branch ${{ github.ref_name }}
      - name: Notify Slack
        uses: 8398a7/action-slack@v3
```

- **Manual Approvals:** Gate high-risk or schema-affecting changes before production.
- **Observability:** Collect Databricks Job metrics (duration, error rates) via the Metrics API into dashboards and set alerts on anomalies.

---

## 8. Why We Avoid Wheel Builds (…for Now)

- **Speed:** Eliminates an extra 30–60s per CI run for packaging.
- **Fidelity:** Code is always sourced directly from the Git-mounted repo.
- **Future-proof:** If external consumers arise, adding a minimal `setup.py` and wheel build is trivial.

---

## 9. Onboarding & Documentation

Provide a top-level `Makefile` and `docs/GETTING_STARTED.md` that guide new engineers through:

1. Forking and cloning the repo.
2. Installing the Databricks CLI and Python dependencies.
3. Configuring secrets in Databricks and local environment variables.
4. Running `make setup` to mount repos and deploy a Dev bundle.
5. Executing smoke tests and viewing pipeline dashboards.

This document ensures first-time contributors can get end-to-end validation in under an hour.

---

## 10. Disaster Recovery – Beyond Code

- **Delta Lake Backups:** Automated daily snapshots to separate storage buckets.
- **Unity Catalog Exports:** Periodic exports of table schemas, permissions, and access policies.
- **DR Runbooks:** Step-by-step instructions for restoring both compute and data in a new workspace.

This comprehensive DR plan covers code, configuration, and data, ensuring business continuity in worst-case scenarios.

---

**Closing Note**\
By weaving together clear repository conventions, automated dependency management, siloed environments, multi-layered testing, security scans, atomic rollbacks, and thorough documentation, we deliver an end-to-end DevOps pipeline that leverages the full power of Databricks while minimizing risk and operational overhead. Teams familiar with this blueprint will be able to innovate rapidly and rely on consistent, predictable deployments.

---

## Conclusion

Thank you for reviewing the Databricks-Native DevOps Blueprint. This document outlines a comprehensive, coherent workflow designed to empower teams to deliver reliable, reproducible, and secure data pipelines on Databricks. By adhering to these practices, your organization can ensure efficient collaboration, rapid iteration, and robust production operations. We hope this serves as a valuable reference and guide for your DevOps journey with Databricks.

