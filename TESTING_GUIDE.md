# Testing Guide

This guide defines how to write, structure, and run tests for this project. Following these standards ensures your Pull Requests pass CI on the first try and that the test suite remains maintainable as the codebase grows.

---

## 1. Testing Standards & Naming Conventions

`pytest` uses **auto-discovery** based on naming conventions. Follow these rules exactly, or your tests will silently not run.

| Item | Convention | Example |
|---|---|---|
| Test files | Must start with `test_` or end with `_test.py` | `test_operations.py` |
| Test functions | Must start with `test_` | `def test_add_returns_correct_sum():` |
| Test classes (optional grouping) | Must start with `Test` and contain **no** `__init__` method | `class TestMatrixMultiplication:` |
| Fixtures | Descriptive noun, no `test_` prefix | `sample_tensor`, `db_connection` |
| Directory | All tests live under `tests/`, mirroring the structure of `src/` | `src/operations.py` → `tests/test_operations.py` |

**`pytest.ini`** (place at repository root) enforces discovery rules explicitly:

```ini
[pytest]
minversion = 8.0
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
addopts = -ra -q --strict-markers
markers =
    slow: marks tests as slow-running (deselect with '-m "not slow"')
    tensor: marks tests that validate tensor/array shape logic
```

---

## 2. Writing Effective Tests

Below are complete, runnable examples covering the four categories every numerical/tensor codebase must test.

Assume the following production code exists in `src/operations.py`:

```python
# src/operations.py
import numpy as np


def add(a: float, b: float) -> float:
    """Return the sum of two numbers."""
    return a + b


def divide(a: float, b: float) -> float:
    """Divide a by b. Raises ValueError if b is zero."""
    if b == 0:
        raise ValueError("Division by zero is not allowed.")
    return a / b


def matmul(a: np.ndarray, b: np.ndarray) -> np.ndarray:
    """Multiply two matrices, validating shape compatibility first."""
    if a.shape[-1] != b.shape[0]:
        raise ValueError(
            f"Shape mismatch: cannot multiply {a.shape} by {b.shape}"
        )
    return np.matmul(a, b)


def normalize(vector: np.ndarray) -> np.ndarray:
    """Normalize a vector to unit length."""
    norm = np.linalg.norm(vector)
    if norm == 0:
        raise ValueError("Cannot normalize a zero vector.")
    return vector / norm
```

### 2.1 Testing Normal Values

```python
# tests/test_operations.py
from src.operations import add


def test_add_returns_correct_sum():
    result = add(2, 3)
    assert result == 5


def test_add_with_negative_numbers():
    result = add(-4, -6)
    assert result == -10
```

### 2.2 Testing Edge Cases

```python
def test_add_with_zero():
    assert add(0, 0) == 0


def test_add_with_large_numbers():
    assert add(1_000_000_000, 1) == 1_000_000_001


def test_divide_with_dividend_zero():
    from src.operations import divide
    assert divide(0, 5) == 0
```

### 2.3 Testing Floating-Point Precision

Never use `==` for floating-point comparisons. Use `pytest.approx` to account for binary floating-point representation error.

```python
import pytest
from src.operations import divide


def test_divide_floating_point_precision():
    result = divide(1, 3)
    # 1/3 = 0.333... cannot be represented exactly in binary floating point.
    assert result == pytest.approx(0.3333333333, rel=1e-9)


def test_add_floating_point_accumulation():
    from src.operations import add
    result = add(0.1, 0.2)
    # 0.1 + 0.2 == 0.30000000000000004 in raw IEEE-754 arithmetic
    assert result == pytest.approx(0.3)
```

### 2.4 Testing Raised Exceptions

```python
import pytest
from src.operations import divide, matmul, normalize
import numpy as np


def test_divide_by_zero_raises_value_error():
    with pytest.raises(ValueError, match="Division by zero is not allowed."):
        divide(10, 0)


def test_matmul_shape_mismatch_raises_value_error():
    a = np.ones((2, 3))
    b = np.ones((4, 5))  # Incompatible: a.shape[-1]=3 != b.shape[0]=4
    with pytest.raises(ValueError, match="Shape mismatch"):
        matmul(a, b)


def test_normalize_zero_vector_raises_value_error():
    zero_vector = np.zeros(3)
    with pytest.raises(ValueError, match="Cannot normalize a zero vector."):
        normalize(zero_vector)
```

---

## 3. Advanced Pytest Patterns

### 3.1 Data-Driven Testing with `@pytest.mark.parametrize`

Instead of writing 5 nearly-identical test functions, parametrize a single test with multiple input/output pairs. This is the recommended pattern for validating mathematical logic across a wide input space.

```python
import pytest
from src.operations import add, divide


@pytest.mark.parametrize(
    "a, b, expected",
    [
        (2, 3, 5),
        (-1, 1, 0),
        (0, 0, 0),
        (100, -50, 50),
        (2.5, 2.5, 5.0),
    ],
)
def test_add_parametrized(a, b, expected):
    assert add(a, b) == pytest.approx(expected)


@pytest.mark.parametrize(
    "a, b",
    [
        (10, 0),
        (-5, 0),
        (0, 0),
    ],
)
def test_divide_by_zero_parametrized(a, b):
    with pytest.raises(ValueError):
        divide(a, b)
```

### 3.2 Reusable Setup Logic with Fixtures

Fixtures eliminate duplicated setup code and are the standard way to provide shared test data (like tensors/matrices) across multiple test functions.

```python
# tests/conftest.py
import numpy as np
import pytest


@pytest.fixture
def identity_matrix_3x3():
    """A reusable 3x3 identity matrix for matrix operation tests."""
    return np.identity(3)


@pytest.fixture
def sample_tensor_batch():
    """A reusable batch of shape (batch=4, height=8, width=8, channels=3)."""
    return np.random.default_rng(seed=42).random((4, 8, 8, 3))


@pytest.fixture(params=[(2, 2), (3, 3), (5, 5)])
def square_matrix(request):
    """Parametrized fixture: yields multiple square matrices of different sizes."""
    rows, cols = request.param
    return np.ones((rows, cols))
```

```python
# tests/test_operations.py
from src.operations import matmul


def test_matmul_with_identity_matrix(identity_matrix_3x3):
    result = matmul(identity_matrix_3x3, identity_matrix_3x3)
    assert result.shape == (3, 3)


def test_tensor_batch_shape_is_preserved(sample_tensor_batch):
    assert sample_tensor_batch.shape == (4, 8, 8, 3)


def test_square_matrix_is_square(square_matrix):
    # This single test automatically runs 3 times, once per fixture param
    assert square_matrix.shape[0] == square_matrix.shape[1]
```

---

## 4. Code Coverage: Measuring and Enforcing

### 4.1 Measuring line coverage locally

```bash
pytest --cov=src --cov-report=term-missing
```

Sample output:

```
Name                    Stmts   Miss  Cover   Missing
-----------------------------------------------------
src/__init__.py             0      0   100%
src/operations.py          24      2    92%   47-48
src/utils.py                12      0   100%
-----------------------------------------------------
TOTAL                       36      2    94%
```

The `Missing` column shows the **exact line numbers not exercised by any test** — use this to find gaps before submitting a PR.

### 4.2 Generating an HTML coverage report (for deeper inspection)

```bash
pytest --cov=src --cov-report=html
open htmlcov/index.html   # macOS
# or: xdg-open htmlcov/index.html   # Linux
```

### 4.3 Enforcing a minimum coverage threshold in CI

The CI workflow (`.github/workflows/test.yml`, see `ARCHITECTURE.md`) enforces this automatically:

```bash
pytest --cov=src --cov-report=term-missing --cov-fail-under=85
```

If total coverage falls below **85%**, `pytest` exits with a non-zero status code, which causes the GitHub Actions job — and therefore the PR check — to fail.

### 4.4 Team Coverage Policy

| Coverage Range | Status |
|---|---|
| ≥ 90% | ✅ Excellent — no action needed |
| 85% – 89% | ⚠️ Acceptable, but flag in PR description which lines are untested and why |
| < 85% | ❌ CI blocks merge — add tests before requesting review |

New files/modules should aim for **95%+** coverage on introduction, since it is far cheaper to write tests alongside new code than to retrofit them later.
