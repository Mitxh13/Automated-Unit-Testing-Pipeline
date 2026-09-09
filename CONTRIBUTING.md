# Contributing Guide

Thank you for contributing to this project. This document defines the workflow every engineer on the team follows — from branching to merge — to keep `main` stable and the CI pipeline fast and trustworthy.

---

## 1. Developer Workflow

### 1.1 Branch Naming Strategy

All work happens on a branch cut from an up-to-date `main`. Branch names must follow this pattern:

```
<type>/<short-description>
```

| Type | Use for | Example |
|---|---|---|
| `feature/` | New functionality | `feature/add-tensor-broadcasting` |
| `fix/` | Bug fixes | `fix/matmul-shape-validation` |
| `hotfix/` | Urgent production fixes | `hotfix/divide-by-zero-crash` |
| `refactor/` | Non-behavioral code restructuring | `refactor/simplify-normalize-fn` |
| `test/` | Test-only additions (no production code change) | `test/add-edge-case-coverage` |
| `docs/` | Documentation-only changes | `docs/update-testing-guide` |

Use lowercase, hyphen-separated words. Do not include ticket numbers unless your team's tracker requires it (e.g., `feature/JIRA-123-tensor-broadcasting`).

```bash
git checkout main
git pull origin main
git checkout -b feature/add-tensor-broadcasting
```

### 1.2 Pull Request Lifecycle

1. **Create the branch** from the latest `main` (see above).
2. **Commit early and often** with clear, atomic commits (see §1.3).
3. **Push the branch** and open a Pull Request targeting `main`.
4. **Fill out the PR template** — description, motivation, and testing performed.
5. **Wait for the automated pipeline** (`Run Pytest Suite`) to complete.
6. **Request review** from at least one teammate (required by branch protection).
7. **Address review comments** by pushing additional commits to the same branch — the PR and CI run update automatically.
8. **Squash and merge** once approved and all checks are green.
9. **Delete the branch** after merge (GitHub can do this automatically — enable "Automatically delete head branches" in repo settings).

### 1.3 Commit Message Standards

Use the **Conventional Commits** format for clean, greppable history:

```
<type>(<scope>): <short summary>

<optional longer description>
```

Examples:

```
feat(operations): add support for batched matrix multiplication
fix(operations): raise ValueError on mismatched tensor shapes
test(operations): add parametrized tests for normalize()
docs(readme): update quickstart instructions for Python 3.10
```

Keep each commit focused on a single logical change — avoid bundling unrelated fixes into one commit.

---

## 2. Pre-Submission PR Checklist

Before opening (or re-requesting review on) a Pull Request, confirm **every** item below. This checklist mirrors exactly what CI will check — completing it locally guarantees a clean pipeline run.

```markdown
### PR Checklist

- [ ] Branch is up to date with `main` (`git pull origin main` merged/rebased in)
- [ ] All local tests pass: `pytest`
- [ ] Coverage meets the 85% threshold: `pytest --cov=src --cov-fail-under=85`
- [ ] Code is formatted: `black src/ tests/`
- [ ] Code is linted with no errors: `flake8 src/ tests/`
- [ ] New functions/classes have docstrings
- [ ] New logic has corresponding tests (normal values, edge cases, exceptions)
- [ ] No commented-out code or debug print statements left in the diff
- [ ] Commit history is clean and atomic (squash fixup commits if needed)
- [ ] PR description explains *what* changed and *why*
```

### Running the full local pre-flight check in one command

```bash
black src/ tests/ && \
flake8 src/ tests/ && \
pytest --cov=src --cov-report=term-missing --cov-fail-under=85
```

If this command exits with code `0`, your PR will pass CI.

---

## 3. Debugging CI Failures

When the `Run Pytest Suite` check fails on GitHub, follow this process rather than guessing.

### Step 1 — Open the failed check

On the PR page, click the red ❌ next to **Run Pytest Suite**, then click **Details** to open the full GitHub Actions log.

### Step 2 — Identify which step failed

The log is broken into collapsible steps matching the workflow file (`Checkout`, `Install dependencies`, `Run pytest with coverage enforcement`, etc.). Expand the **first** step showing a red ❌ — that is where the failure originates. Ignore later steps; they are often downstream noise.

### Step 3 — Read the pytest failure output

Look for the `FAILED` summary block near the end of the `Run pytest with coverage enforcement` step, e.g.:

```
FAILED tests/test_operations.py::test_matmul_shape_mismatch_raises_value_error
    - assert ValueError not raised
=================== 1 failed, 42 passed in 3.21s ===================
```

This tells you exactly which test function failed and why (assertion mismatch, unhandled exception, etc.).

### Step 4 — Reproduce the exact failure locally

Run the *specific* failing test with verbose output:

```bash
pytest tests/test_operations.py::test_matmul_shape_mismatch_raises_value_error -v
```

If it passes locally but fails in CI, check for **environment drift**:

```bash
# Recreate a clean environment identical to CI
rm -rf venv
python3.10 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt -r requirements-dev.txt
pytest
```

This eliminates the most common cause of "works on my machine but fails in CI" — a stale local virtual environment with mismatched package versions.

### Step 5 — Check for coverage failures specifically

If tests pass but the job still fails, check the tail of the log for:

```
FAIL Required test coverage of 85% not reached. Total coverage: 81.40%
```

Run coverage locally to find exactly which lines are untested:

```bash
pytest --cov=src --cov-report=term-missing
```

Add tests for the reported missing line numbers, then re-run the full suite.

### Step 6 — Check for flaky/non-deterministic tests

If a test fails intermittently (passes on re-run with no code changes), it likely depends on:
- Unseeded randomness (`np.random.random()` instead of a seeded `np.random.default_rng(seed=...)`)
- Test execution order/shared mutable state between tests
- Real time/timestamps

Fix by seeding all randomness in fixtures (see `TESTING_GUIDE.md` §3.2) and ensuring each test is fully independent.

### Step 7 — Re-push and confirm

Once fixed, push the commit. The workflow re-triggers automatically on the same PR:

```bash
git add .
git commit -m "fix(operations): correct shape validation to raise ValueError"
git push origin feature/add-tensor-broadcasting
```

Wait for the check to turn green, then proceed with merge per §1.2.

---

## 4. Getting Help

If you're stuck on a CI failure for more than 15–20 minutes, don't keep guessing — post the failing job's log link in the team channel. A second set of eyes on the exact error output is almost always faster than isolated debugging, especially for tensor shape errors that can originate several function calls upstream of the assertion.
