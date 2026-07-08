# Solution

An installable Python distribution is described by `pyproject.toml` — a `[build-system]` table naming the build backend and a `[project]` table holding the metadata — from which `python3 -m build` produces a wheel under `dist/`. In this task you write `pytest` unit tests for the fraud-detection module's `predict()` function, then fix the draft `pyproject.toml` (adding the build-system section, correcting the name and version, pinning `requires-python`, declaring dependencies, and setting `pythonpath = ["src"]`) so the tests pass and the build emits a `fraud_detection-0.1.0` wheel.

> As an MLOps engineer, you get the module test-covered and packaged into a reproducible, installable wheel for the deployment team — the prediction logic itself is given and left unchanged.

#### Follow the steps below

**About unit testing with `pytest`:** `pytest` discovers files named `test_*.py`, runs every function named `test_*`, and fails any `assert` that does not hold. Testing a module before you ship it is how you prove it behaves as intended. The tests import the module from `src/`; the `pythonpath = ["src"]` setting under `[tool.pytest.ini_options]` in `pyproject.toml` tells pytest where to find it.

**About Python packaging:** An installable Python distribution is described by `pyproject.toml`. The `[build-system]` table tells the builder which backend to use (here `setuptools`); the `[project]` table holds the metadata — `name`, `version`, `requires-python`, and runtime `dependencies`. `python3 -m build` reads this file and writes a *wheel* (`.whl`, the installable binary format) plus a source distribution (`.tar.gz`) to `dist/`. The wheel filename encodes the normalised name and version (e.g. `fraud_detection-0.1.0-py3-none-any.whl`).

##### 1. Inspect the module and the draft configuration.
Review the module you will test and package, and compare the `pyproject.toml` against the requirements in the task.
```
cd /root/code/fraud-detection
cat src/fraud_detection/predict.py
cat pyproject.toml
```
`predict()` returns `1` for any row whose first value (the amount) exceeds `100`, and `0` otherwise. The `pyproject.toml` has no `[build-system]` section, declares the distribution name with a hyphen, pins the version to `0.0.1`, requires Python `>=3.8`, ships an empty `dependencies` list, and has no pytest configuration.

##### 2. Correct `pyproject.toml`.
Replace the file so it satisfies the packaging requirements and lets pytest import the package from `src/`.
```
cat > pyproject.toml << 'TOMLEOF'
[build-system]
requires = ["setuptools>=61.0", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "fraud_detection"
version = "0.1.0"
description = "Fraud detection model for xFusionCorp Industries"
requires-python = ">=3.10"
dependencies = [
    "scikit-learn",
    "pandas",
    "numpy",
]

[tool.setuptools.packages.find]
where = ["src"]

[tool.pytest.ini_options]
pythonpath = ["src"]
TOMLEOF
```

##### 3. Write unit tests for the module.
Create `tests/test_predict.py` covering a fraudulent row (amount above 100) and a legitimate row (amount at or below 100). Each test imports `predict` and asserts on its output.
```
cat > tests/test_predict.py << 'PYEOF'
from fraud_detection import predict


def test_flags_high_amount():
    assert predict([[150.0]]) == [1]


def test_passes_low_amount():
    assert predict([[50.0]]) == [0]
PYEOF
```
Run the tests from the project directory — both must pass.
```
python3 -m pytest tests/ -q
```

##### 4. Build the package.
With the configuration corrected, build the source distribution and wheel.
```
python3 -m build
```

##### 5. Verify.
Confirm the tests pass and a correctly-named wheel is present.
```
python3 -m pytest tests/ -q
ls dist/
```
`pytest` reports all tests passing, and `dist/` contains a `.whl` that starts with `fraud_detection-0.1.0`. (Any earlier `fraud_detection-0.0.1` artefacts from a prior build may remain alongside it and do not affect validation.)

---

**References:**
- [pytest — documentation](https://docs.pytest.org/en/stable/)
- [Python Packaging — Writing your `pyproject.toml`](https://packaging.python.org/en/latest/guides/writing-pyproject-toml/)
- [Python Packaging — Packaging Python Projects (build a wheel)](https://packaging.python.org/en/latest/tutorials/packaging-projects/)
