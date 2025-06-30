Databricks-Native DevOps Blueprint

A fully “Databricks-only” approach—no local environment mirroring, no in-repo wheel packaging—yet enterprise-grade in security, rollback, CI/CD, and consistency.

⸻

1. Repository Layout – Why We Organize This Way

What happens: When a developer opens the repository they see a consistent, opinionated folder structure.
Why it’s needed: Predictability accelerates onboarding and reduces cognitive overhead. Clear separation of code, notebooks, configs, and tests prevents accidental misuse of development artifacts in production.
How it fits: A well-structured repo lays the foundation for reproducible builds, CI enforcement, and environment isolation.

your-project/
├── deep_project_core/        ← Pure-Python libraries (portable business logic)
├── notebooks/                ← All notebooks (EDA, workflows, smoke tests)
│   ├── eda/
│   ├── workflows/            ← Each notebook maps 1:1 to a Job
│   └── smoke_tests/
├── requirements/             ← Dependency inputs & outputs
│   ├── runtime-baseline.in          # DBR 16.4 ML exact snapshot
│   ├── runtime-baseline.constraints # same snapshot → “≤” constraints
│   ├── additional-packages.in       # direct deps
│   ├── requirements-combined.in     # baseline + additions (for conflict checks)
│   └── requirements-final.txt       # fully-pinned install list
├── init/                     ← Cluster init scripts
│   └── setup-env.sh
├── tests/                    ← Unit, integration & smoke artifacts
│   ├── unit/
│   ├── integration/
│   └── data/
├── config/                   ← Job JSON templates, env configs, secrets mapping
├── databricks.yml            ← Deployment manifest (jobs, clusters, init scripts)
└── .github/workflows/ci.yml  ← CI pipeline definition



⸻

2. Dependency Workflow – Ensuring Reproducible, Evolvable Environments

What happens: We capture the DBR baseline, constrain it, declare our additions, resolve a fully-pinned graph, install it on clusters, and then use our tests to verify any evolution of those constraints.

2.1 Snapshot the Exact DBR Baseline

Run once per Runtime upgrade in a clean cluster:

import pkg_resources, datetime, pathlib

entries = sorted(f"{p.project_name}=={p.version}"
                 for p in pkg_resources.working_set)
header = f"# DBR16.4 ML snapshot {datetime.datetime.utcnow().isoformat()} UTC\n\n"
pathlib.Path("requirements/runtime-baseline.in") \
        .write_text(header + "\n".join(entries) + "\n")

Why: Guarantees we know exactly what Databricks ships, so no accidental upgrades or downgrades.

2.2 Convert to “≤” Constraints

Turn every pkg==ver into pkg<=ver so pip-compile cannot pick above the DBR ceiling:

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

Why: Enforces DBR’s maxima but still allows pip-compile to choose lower versions if needed.

2.3 Declare Direct Dependencies

Edit requirements/additional-packages.in with only your new direct libraries (e.g. pandas==2.2.0).

2.4 Resolve & Pin with pip-compile

cd requirements/
pip install --quiet pip-tools
pip-compile \
  additional-packages.in \
  --constraint=runtime-baseline.constraints \
  --output-file=requirements-final.txt \
  --allow-unsafe

What: Merges your direct deps with the DBR constraint ceiling, catches any conflicts, and pins every transitive version.
Why: Fails fast in CI if a new library demands a version above what DBR provides—forcing a conscious decision.

2.5 Evolving the Baseline: Live Constraint-Bumping Process

When pip-compile errors on a new library:
	1.	Developer Workflow on feature branch:
	•	Install new_lib interactively in Dev mount (%pip install new_lib) and verify code imports.
	•	Add new_lib==X.Y.Z to additional-packages.in.
	2.	Run pip-compile (as above).
	•	If it fails, decide to either upgrade DBR (regenerate runtime-baseline.in & .constraints from a new 17.x cluster) or pin/replace the new library.
	3.	Run Tests on Dev cluster:

%pip install -r requirements/requirements-final.txt
pytest tests/unit
pytest tests/integration


	4.	Open a PR including:
	•	Updated additional-packages.in
	•	Regenerated requirements-final.txt
	•	(If you bumped the ceiling) new runtime-baseline.in & .constraints
	5.	CI re-validates pip-compile, unit, integration, and security scans.
	6.	Merge & Promote → dev deploy → smoke tests → staging/prod as usual.

Why: Your test suite becomes the signal that it’s safe to evolve the DBR ceiling, and every bump is fully audited in Git.

⸻

3. Cluster Roles & Repo Mounts – Siloed Environments in One Workspace

Path	Git Ref	Purpose
/Repos/...-dev	main or feature/*	Developer sandbox & QA
/Repos/...-stg	release/x.y-RC	Pre-prod validation
/Repos/...	release/x.y	Production pipelines

	•	Asset Bundles deploy a specific branch into each mount without changing job definitions.
	•	Branch promotion uses CLI or GitHub Actions:

databricks bundle deploy --target staging --branch release/x.y-RC
databricks bundle deploy --target prod    --branch release/x.y



⸻

4. Testing Pyramid – Multi-Layered Validation
	1.	Unit tests (tests/unit/): fast, pure-Python, function-level checks.
	2.	Integration tests (tests/integration/): SparkSession + tiny JSON fixtures → end-to-end pipeline checks.
	3.	Smoke tests (notebooks/smoke_tests/): run actual Job notebooks via pytest wrapper, validating init scripts, cluster configs, and entry-point correctness.

Why: Each layer catches a different class of error—from logic mistakes to environment misconfigurations—before any code reaches production.

⸻

5. Security & Compliance – Automated Scans Everywhere

Tool	Purpose	CI Command
pip-audit	CVE + license metadata	pip-audit --require-licenses -r requirements-final.txt
safety	PyPI vulnerability database	safety check --file requirements-final.txt
pip-licenses	Detect disallowed open-source licenses	pip-licenses --from=requirements-final.txt --fail-on="GPL;LGPL"

	•	CI enforcement: Any scanner failure blocks the merge.
	•	Weekly live scans: Scheduled Databricks Job re-runs the same scanners against running clusters, posting new findings to Slack.

⸻

6. Rollback Strategy – Fast, Atomic Recovery
	•	Keep two stable release branches (current + previous).
	•	Rollback command:

databricks bundle deploy --target prod --branch release/x.y-1

	•	Automated drills run quarterly in sandbox, verify rollback & forward-deploy via smoke tests.
	•	Canary deployments: For critical pipelines, run 5–10% of prod traffic on the new release first, monitor success, then cut over fully.

Why: Rapid, reliable recovery minimizes MTTR during outages.

⸻

7. CI/CD Workflow – Gatekeeping & Observability

jobs:
  unit:
    runs-on: ubuntu-latest
    steps:
      - checkout, setup Python
      - pytest tests/unit
      - security scans

  integration:
    needs: unit
    steps:
      - databricks bundle deploy --target dev --branch ${{ github.head_ref }}
      - trigger & poll Dev smoke job → fail on non-SUCCESS

  deploy-prod:
    needs: [unit, integration]
    if: startsWith(github.ref, 'refs/heads/release/')
    steps:
      - approval
      - databricks bundle deploy --target prod --branch ${{ github.ref_name }}
      - slack notification
