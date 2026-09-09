# Pipeline Architecture

This document explains the internal design of the Automated Unit Testing Pipeline: what each part of the GitHub Actions workflow does, how isolation is guaranteed, how the pipeline is enforced via branch protection, and how it scales as the team and codebase grow.

---

## 1. The Complete Workflow File

Below is the full production-ready workflow. Save this exact content at:

```
.github/workflows/test.yml
```

```yaml
name: Automated Unit Testing Pipeline

on:
  pull_request:
    branches: [ "main" ]
  push:
    branches: [ "main" ]

permissions:
  contents: read
  pull-requests: write

concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  test:
    name: Run Pytest Suite
    runs-on: ubuntu-latest
    timeout-minutes: 15

    steps:
      - name: Checkout repository code
        uses: actions/checkout@v4

      - name: Set up Python 3.10
        uses: actions/setup-python@v5
        with:
          python-version: "3.10"
          cache: "pip"

      - name: Create virtual environment
        run: python -m venv venv

      - name: Install dependencies
        run: |
          source venv/bin/activate
          pip install --upgrade pip
          pip install -r requirements.txt
          pip install -r requirements-dev.txt

      - name: Run pytest with coverage enforcement
        run: |
          source venv/bin/activate
          pytest \
            --cov=src \
            --cov-report=term-missing \
            --cov-report=xml \
            --cov-fail-under=85 \
            --junitxml=pytest-results.xml

      - name: Upload test results artifact
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: pytest-results
          path: |
            pytest-results.xml
            coverage.xml

      - name: Publish coverage summary to job log
        if: always()
        run: |
          source venv/bin/activate
          coverage report --skip-covered
```

---

## 2. Line-by-Line Breakdown

| Section | Line(s) | Purpose |
|---|---|---|
| `name:` | 1 | Human-readable name shown in the GitHub Actions UI tab. |
| `on:` | 3–6 | Defines triggers. This workflow runs on every `pull_request` targeting `main` (the primary gate) and on direct `push` events to `main` (safety net for hotfixes/admin merges). |
| `permissions:` | 8–10 | Grants the workflow the **minimum necessary** GitHub token permissions — read-only repo access plus write access to PR comments/checks. This follows the principle of least privilege. |
| `concurrency:` | 12–14 | If a developer pushes multiple commits to the same PR in quick succession, this cancels the older, now-outdated run (`cancel-in-progress: true`) so runners aren't wasted on stale code. |
| `jobs.test.runs-on` | 18 | Specifies the OS image for the runner VM — `ubuntu-latest`, a clean Linux environment maintained by GitHub. |
| `timeout-minutes: 15` | 19 | Hard safety limit. If the test suite hangs (e.g., an infinite loop introduced by a bug), the job is killed after 15 minutes instead of consuming runner minutes indefinitely. |
| `actions/checkout@v4` | 22–23 | Clones the exact commit of the PR branch into the runner's filesystem. Without this step, the runner has no access to your code at all. |
| `actions/setup-python@v5` | 25–29 | Installs Python 3.10 onto the runner and enables **pip caching**, so dependency installs are faster on subsequent runs. |
| `Create virtual environment` | 31–32 | Isolates Python dependencies for this job from anything pre-installed on the runner image. |
| `Install dependencies` | 34–39 | Activates the venv and installs pinned versions from both `requirements.txt` (runtime) and `requirements-dev.txt` (testing tools: `pytest`, `pytest-cov`). |
| `Run pytest with coverage enforcement` | 41–48 | The core quality gate. `--cov-fail-under=85` causes pytest to **exit with a non-zero status code** if line coverage falls under 85%, which GitHub Actions interprets as a failed job. |
| `Upload test results artifact` | 50–56 | Preserves the JUnit XML and coverage XML reports as downloadable artifacts attached to the workflow run, even if the job fails (`if: always()`), so engineers can inspect exactly what broke. |
| `Publish coverage summary` | 58–62 | Prints a human-readable coverage table directly into the Actions log for a fast, no-download-required view. |

---

## 3. Environment Isolation

Every workflow run executes in a **brand-new, ephemeral virtual machine**. This is a foundational reliability guarantee of GitHub Actions and is essential for trustworthy CI results.

**Key isolation properties of `ubuntu-latest` runners:**

- **Fresh filesystem every run** — the VM image is provisioned from a clean snapshot; there is no shared state, cache pollution, or "works on my machine" drift between runs.
- **No cross-run contamination** — environment variables, installed packages, or file writes from a previous PR's run cannot leak into the current run.
- **Deterministic dependency resolution** — because dependencies are installed fresh from `requirements.txt` inside a fresh `venv`, the pipeline validates the *exact* dependency graph a new developer would get, not a cached or manually-patched local environment.
- **Automatic teardown** — after the job completes (pass or fail), the VM is destroyed. Nothing persists except the artifacts you explicitly upload (e.g., `pytest-results.xml`).
- **Resource guarantee** — GitHub provisions each `ubuntu-latest` runner with a standard, consistent amount of CPU/RAM (2-core, 7GB RAM as of standard hosted runners), which means performance-sensitive tensor-shape tests behave consistently run over run.

This isolation is what allows the team to trust a green checkmark: it proves the code works in a controlled, reproducible environment — not just "on someone's laptop."

---

## 4. Branch Protection & Enforcement

A workflow that merely *runs* tests is not enough — it must be **wired into GitHub's Branch Protection Rules** so that a failing test *physically prevents* a merge. Configure this once per repository:

### Step-by-step setup

1. Navigate to your repository → **Settings** → **Branches**.
2. Under **Branch protection rules**, click **Add rule** (or edit the existing rule for `main`).
3. Set **Branch name pattern** to `main`.
4. Enable **Require a pull request before merging**.
   - Optionally enable **Require approvals** (recommended: 1 approval minimum for a 4-person team).
5. Enable **Require status checks to pass before merging**.
   - In the search box, find and select the job name exactly as it appears in the workflow: **`Run Pytest Suite`**.
6. Enable **Require branches to be up to date before merging** — this forces the PR branch to be rebased/merged with the latest `main` before the check is considered valid, preventing "it passed on an old version of main" false positives.
7. (Recommended) Enable **Do not allow bypassing the above settings**, so even repository admins are held to the same standard unless explicitly overridden.
8. Click **Save changes**.

### Result

Once configured, the GitHub PR UI will display:

```
❌ Some checks were not successful
1 failing check: Run Pytest Suite — This check failed
Merge blocked
```

The green **"Merge pull request"** button becomes visually disabled and non-clickable until the `Run Pytest Suite` check reports success. This is enforced server-side by GitHub — it cannot be bypassed by force-pushing, closing/reopening the PR, or local Git tricks.

---

## 5. Scalability Roadmap

As the team and codebase grow over the 15–16 month lifecycle, the pipeline is designed to expand along the following path without requiring a rewrite.

### Phase 1 (Current) — Single-version correctness gate
- Single Python 3.10 runner
- `pytest` + `pytest-cov` with an 85% threshold
- Runs on every PR to `main`

### Phase 2 — Matrix Testing Across Python Versions

Expand the `test` job into a **build matrix** to guarantee compatibility across the versions your team may eventually adopt (e.g., ahead of a Python upgrade):

```yaml
jobs:
  test:
    name: Run Pytest Suite (Python ${{ matrix.python-version }})
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        python-version: ["3.10", "3.11", "3.12"]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
          cache: "pip"
      - run: pip install -r requirements.txt -r requirements-dev.txt
      - run: pytest --cov=src --cov-fail-under=85
```

`fail-fast: false` ensures that a failure on Python 3.12 doesn't cancel the 3.10 and 3.11 runs — you get full visibility into exactly which versions are affected.

### Phase 3 — Static Security & Code Quality Scanning

Add a parallel job for static analysis so security issues are caught alongside logic errors:

```yaml
  security-scan:
    name: Static Security Analysis
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.10"
      - name: Install security tools
        run: pip install bandit safety
      - name: Run Bandit (static code security scan)
        run: bandit -r src/ -ll
      - name: Run Safety (dependency vulnerability scan)
        run: safety check -r requirements.txt
```

- **Bandit** scans Python source code for common security anti-patterns (e.g., use of `eval`, hardcoded secrets, insecure deserialization).
- **Safety** cross-references your pinned dependency versions against known CVE databases.

### Phase 4 — Performance Regression Detection

For tensor-heavy operations, introduce a benchmarking job using `pytest-benchmark` that fails the build if any operation regresses beyond an acceptable threshold (e.g., >20% slower than the `main` baseline), protecting against silently introduced performance bugs alongside correctness bugs.

### Phase 5 — Required Checks Expansion

As Phases 2–4 are added, update the Branch Protection Rule (Section 4) to require **all** new job names as required status checks, ensuring `main` stays protected as the pipeline's scope grows.

