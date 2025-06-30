Databricks-Native DevOps Blueprint

A fully “Databricks-only” approach—no local environment mirroring, no in-repo wheel packaging—yet enterprise-grade in security, rollback, CI/CD, and consistency.

## 1. Repository Layout – Why We Organize This Way

**What happens:** When a developer opens the repository, they see a consistent, opinionated folder structure.\
**Why it’s needed:** Predictability accelerates onboarding and reduces cognitive overhead. Clear separation of code, notebooks, configs, and tests prevents accidental misuse of development artifacts in production.\
**How it fits:** A well-structured repo lays the foundation for reproducible builds, CI enforcement, and environment isolation.

```text
your-project/
├── deep_project_core/        ← Pure Python libraries (portable business logic)
├── notebooks/                ← All notebooks (EDA, workflows, smoke tests)
    ├── eda/ 
    ├── workflows/ 
├── requirements/
│   ├── runtime-baseline-ml.in      ← DBR XX.X ML snapshot
│   ├── additional-packages-ml.in   ← direct deps
│   ├── requirements-combined-ml.in ← baseline + additions
│   └── requirements-ml.txt   ← locked full graph
│   ├── runtime-baseline.in      ← DBR XX.X ML snapshot
│   ├── additional-packages.in   ← direct deps
│   ├── requirements-combined.in ← baseline + additions
│   └── requirements.txt   ← locked full graph
│   ├── runtime-baseline-gpu.in      ← DBR XX.X ML snapshot
│   ├── additional-packages-gpu.in   ← direct deps
│   ├── requirements-combined-gpu.in ← baseline + additions
│   └── requirements-gpu.txt   ← locked full graph
├── init/                     ← Cluster initialization scripts
├── tests/                    ← Unit, integration, and smoke-test artifacts
│   ├── unit/                    ← pytest unit tests (1:test per module)
│   ├── integration/             ← pytest Spark-backed pipeline tests
│   └── data/                    ← small JSON/CSV fixtures
├── config/                   ← Environment configs (job JSON templates, secrets mapping)
├── databricks.yml            ← Deployment manifest (jobs, clusters, init scripts)
└── .github/workflows/ci.yml  ← CI pipeline definition
```

- \`\`: Contains reusable functions and classes without any Databricks-specific imports. This separation ensures we can later extract or wrap this code into a microservice, API, or external library without refactoring `dbutils` calls.
- \`\`: Subdivided into `eda/`, `workflows/`, and `smoke_tests/`. Each workflow notebook corresponds 1:1 with a Job in Databricks. Keeping EDA notebooks separate prevents accidental scheduling of exploratory code.
- \`\`: Houses four files:
  1. `runtime-baseline.in`: Auto-generated snapshot of all DBR 16.4 ML pre-installed packages.
  2. `additional-packages.in`: Developer-declared extras.
  3. `requirements-combined.in`: Generated merge of baseline + additions for conflict checking.
  4. `requirements-final.txt`: Fully resolved, pinned graph injected into clusters.
- \`\`: Runs on every cluster startup to uninstall stray packages and reinstall only what `requirements-final.txt` dictates, guaranteeing fidelity to CI-validated environments.
- \`\`:
  - `unit/`: Fast, pure-Python tests.
  - `integration/`: Spark-based mini-pipeline runs on tiny JSON data.
  - `smoke_tests/`: Calls the same pytest suite via notebooks, validating the notebook entry point, job wrapper, and cluster environment.
- \`\`: New addition. Centralizes job JSON templates (parameterized via Jinja), environment-specific values (S3 buckets, endpoints), and secrets-to-key mapping. Having a single source of truth prevents drift.
- \`\`: Declares every artifact and the mapping of jobs to clusters across `dev`, `staging`, and `prod`. Infrastructure-as-code ensures deployment consistency.
- \`\`: Orchestrates CI: unit tests → init scripts → integration & smoke tests → asset bundle deploys.

---

## 2. Dependency Workflow – Ensuring Reproducible Environments

**What happens:** We snapshot DBR’s built-in packages, declare additional needs, merge and pin dependencies, then enforce them at cluster startup.

1. **Snapshot once per DBR version**

   ```python
   import pkg_resources, datetime, pathlib
   baseline = sorted(f"{p.project_name}=={p.version}" for p in pkg_resources.working_set)
   pathlib.Path("requirements/runtime-baseline.in").write_text(
       f"# DBR16.4 ML snapshot {datetime.datetime.utcnow().isoformat()}\n\n" + "\n".join(baseline)
   )
   ```

   *Why:* Captures exactly what Databricks pre-installs, eliminating silent transitive upgrades or downgrades.

2. **Declare project dependencies** in `additional-packages.in` (e.g., `pandas==2.2.0`).

3. \*\*Run \*\*\`\`

   - Merges baseline + additions.
   - Detects version conflicts early (e.g., if DBR’s `joblib` clashes with `scikit-learn`).\
     *Why:* Fails fast in CI before any cluster launches with inconsistent libs.

4. **Cluster startup (**\`\`**\*\*\*\*)**

   ```bash
   pip freeze | xargs pip uninstall -y    # Wipe stray packages
   pip install -r requirements-final.txt  # Reinstall pinned graph
   ```

   *Why:* Ensures every interactive notebook and automated Job runs with the exact, tested dependency graph.

**Extension – Containerized Environments**: For even faster startup and stronger isolation, you can build a custom Docker image via Databricks Container Services that bakes in your pinned dependencies. This moves environment enforcement from scripting into infrastructure, reducing cluster spin-up time by \~30–60s.

---

## 3. Cluster Roles & Repo Mounts – Siloed Environments in a Single Workspace

**What happens:** We maintain three stable repo mounts—`-dev`, `-stg`, `-prod`—all under one Databricks workspace but with distinct Git refs.

| Path                            | Git Ref               | Job Prefix | Purpose                                          |
| ------------------------------- | --------------------- | ---------- | ------------------------------------------------ |
| `/Repos/.../simple-project-dev` | `main` or `feature/*` | `Dev – `   | Daily exploration; breaking changes accepted     |
| `/Repos/.../simple-project-stg` | `release/x.y-RC`      | `Stg – `   | Dress rehearsal; final QA against near-prod data |
| `/Repos/.../simple-project`     | `release/x.y`         | `Prod – `  | Live data; zero tolerance for surprises          |

- **Asset Bundle deployments** (`databricks bundle deploy`) update each mount’s Git branch without touching job definitions.
- **RBAC Segmentation**: Enforce least-privilege by assigning distinct IAM/Databricks permissions to each mount so, for example, Dev users cannot accidentally write to Prod.
- **Environment Promotion CLI**: A simple wrapper (or GitHub Action) automates branch promotion from Dev → Staging → Prod, reducing manual steps and human error.

---

## 4. Testing Pyramid – Multi-Layered Validation

**What happens:** CI runs three layers of tests in sequence: unit → integration → smoke.

1. **Unit tests**
   - Pure-Python, <1s runtime.
   - Catches logical bugs early (e.g. off-by-one errors).
2. **Integration tests**
   - Spin up a SparkSession on the Dev cluster.
   - Run pipelines on mini JSON datasets (5–20 rows).
   - Validate schema, joins, and data transformations.
3. **Smoke notebook tests**
   - Launch the actual Databricks Job wrapper as a notebook.
   - Verifies job definitions, permissions, init scripts, and cluster-scoped configs.

**Additional safeguards:**

- **Property-based testing** with Hypothesis for library code to uncover edge cases.
- **Data drift checks**: Assert on pre-defined distribution thresholds (e.g. null rates, distinct counts) so silent data-quality regressions fail CI.

**Why it’s needed:** Each layer catches a different class of error—from simple logic mistakes to environment misconfigurations—before any code reaches production.

---

## 5. Security & Compliance – Automated Scans Everywhere

**What happens:** We integrate CVE and license scanners at commit-time and weekly in production.

| Tool           | Purpose                                | CI Command                                                        |
| -------------- | -------------------------------------- | ----------------------------------------------------------------- |
| `pip-audit`    | CVE + license metadata                 | `pip-audit --require-licenses -r requirements-final.txt`          |
| `safety`       | PyPI vulnerability database            | `safety check --file requirements-final.txt`                      |
| `pip-licenses` | Detect disallowed open-source licenses | `pip-licenses --from=requirements-final.txt --fail-on="GPL;LGPL"` |

- **CI enforcement**: Any scanner failure blocks the merge.
- **Weekly live scans**: Scheduled Databricks Job re-runs the same scanners against the running cluster, posting new CVEs to Slack.
- **Policy-as-Code**: Embed governance rules (e.g. allowed cluster network settings, isolation policies) using Open Policy Agent (OPA) integrated into CI and cluster startup.
- **Secrets Management**: All tokens and credentials are stored in Databricks Secrets, referenced via `dbutils.secrets.get()`, ensuring no sensitive data in plain text.

**Why it’s needed:** Proactively catch vulnerabilities, license violations, and misconfigurations before they impact production or violate compliance mandates.

---

## 6. Rollback Strategy – Fast Recovery Under Pressure

**What happens:** Keeping two “last known good” release branches (`release/x.y` and `release/x.y-1`) lets us revert prod with one command:

```bash
databricks bundle deploy --target prod --branch release/x.y-1
```

- **Atomic switch**: Jobs reference notebook paths; a bundle deploy rewrites code, init scripts, and requirements in one shot.
- **Job definition store**: We `databricks jobs get --job-id …` and commit the JSON to Git, so a UI deletion or manual tweak can be rehydrated by applying the stored JSON.
- **Automated rollback drills**: A quarterly scheduled GitHub Action triggers a rollback in a sandbox workspace, runs smoke tests, then redeploys forward—validating our recovery runbooks.
- **Canary deployments**: For critical pipelines, run 5–10% of prod traffic on the new release first, monitor success, then cut over fully.

**Why it’s needed:** Rapid, reliable recovery is essential for SRE and incident-response teams, minimizing MTTR during outages.

---

## 7. CI/CD Workflow – Gatekeeping and Observability

**What happens:** GitHub Actions orchestrates a multi-stage pipeline:

1. **Unit & security** → 2. **Dev deploy + smoke tests** → 3. **Staging & Prod deploys** (with approvals)

```yaml
jobs:
  unit:
    runs-on: ubuntu-latest
    steps:
      - checkout, setup Python
      - pytest tests/unit
      - pip-audit, safety, pip-licenses
  integration:
    needs: unit
    steps:
      - databricks bundle deploy --target dev --branch ${{ github.head_ref }}
      - trigger smoke job on Dev cluster; poll success
  deploy-prod:
    needs: [unit, integration]
    if: startsWith(github.ref, 'refs/heads/release/')
    steps:
      - approval: manager
      - databricks bundle deploy --target prod --branch ${{ github.ref_name }}
      - slack notification
```

- **Dynamic approvals**: Require manual sign-off for high-risk or data-impacting jobs.
- **Observability**: Capture job-level metrics (run times, error rates, data volumes) via Databricks Metrics API. Dashboards in Grafana or Databricks SQL monitor SLIs/SLOs and trigger alerts on anomalies.

**Why it’s needed:** Ensures code only advances when all quality gates pass, and lets us track operational health over time.

---

## 8. Why We Avoid Wheel Builds (…for Now)

**What happens:** We import `deep_project_core` directly from the mounted repo rather than building a Python wheel.

- **Speed**: Avoids 30–60s per CI run for wheel creation.
- **Fidelity**: No extra artifacts to version; code is always current in the mounted repo.
- **Future-proof**: If we later expose `deep_project_core` as an external library, adding a minimal `setup.py` and wheel build is trivial.

**Why it’s needed:** Simplifies the pipeline while meeting all current Databricks-only use cases.

---

## 9. Onboarding & Documentation – Getting Up and Running

**What happens:** We provide a `Makefile` and `docs/GETTING_STARTED.md` that guide new developers through:

1. Forking/cloning the repo
2. Installing local CLI tools (`databricks-cli`, Python dependencies)
3. Setting up Databricks Secrets and config variables
4. Running `make setup` to mount repos and deploy a dev bundle
5. Executing smoke tests and viewing dashboards

**Why it’s needed:** Reduces ramp-up time from days to hours and codifies tribal knowledge into repeatable scripts.

---

## 10. Disaster Recovery (DR) – Beyond Code

**What happens:** In addition to code rollback, we ensure data resilience by:

- **Delta Lake backups**: Automated daily snapshots of critical tables to a separate S3/GCS bucket.
- **Unity Catalog exports**: Periodic exports of table schemas and access policies.
- **DR runbooks**: Step-by-step guides for restoring data and metadata in a new workspace.

**Why it’s needed:** A true DR plan covers both code and data, ensuring business continuity even in catastrophic failures.

---

**Closing Note**\
By embedding each stage—code structure, dependency management, environment isolation, multi-layer testing, security scans, rollback capabilities, CI/CD gates, observability, and DR—directly into the Databricks ecosystem, we achieve a seamless, enterprise-grade DevOps pipeline. Developers never leave the platform they know, while operators gain confidence that every production notebook runs exactly the code and dependencies validated in CI.

