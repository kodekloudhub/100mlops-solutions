# Solution

`uv` is a fast Python package manager that compiles a high-level `requirements.in` into a fully pinned `requirements.txt` lockfile, so the exact same environment resolves on any machine. In this task you fix a teammate's malformed `requirements.in` — dropping the deprecated `sklearn` name, removing an unresolvable `mlflow` pin, and adding the missing `pandas` — then run `uv pip compile` to produce a lockfile that pins all four top-level packages plus their transitive dependencies.

> As an MLOps engineer, you turn a loose dependency wishlist into a deterministic, fully pinned lockfile so every machine resolves the same environment — you are not writing model code here.

#### Follow the steps below

**About `uv` and lockfiles:** `uv` is a fast Python package manager. A `requirements.in` lists the *high-level* dependencies you care about; `uv pip compile` resolves them — and all their transitive dependencies — into a fully pinned `requirements.txt` *lockfile*, so the exact same environment is reproducible on any machine. The fix here is to make the input spec valid so the compile can resolve it.

##### 1. Inspect the existing specification.
Review the file that was left behind and compare it against the required dependency list in the task.
```
cd /root/code/fraud-detection
cat requirements.in
```
The file uses the deprecated `sklearn` package name, pins `mlflow` to a version that does not exist on PyPI, and is missing `pandas`.

##### 2. Correct the specification.
Replace the file contents with the four required top-level packages. Remove version pins that cannot be resolved; `uv` will pin the compiled versions in the next step.
```
cat > requirements.in << 'EOF'
# Fraud detection project dependencies
scikit-learn
mlflow
pandas
numpy
EOF
```

##### 3. Compile the lockfile.
`uv pip compile` resolves the full dependency tree and writes a pinned `requirements.txt` that includes every transitive dependency. The resolution runs in seconds, without any solver memory overhead.
```
uv pip compile requirements.in -o requirements.txt
```

##### 4. Verify.
Confirm that the lockfile exists, carries the `uv` autogeneration header, and pins every package to an exact version.
```
head -5 requirements.txt
grep -E '^(scikit-learn|mlflow|pandas|numpy)==' requirements.txt
wc -l requirements.txt
```
The fourth command reports the total number of lines — a healthy lockfile contains far more than the four top-level packages because `uv` pins every transitive dependency as well.

---

**References:**
- [uv — `uv pip compile` (locking requirements)](https://docs.astral.sh/uv/pip/compile/)
- [pip — requirements file format](https://pip.pypa.io/en/stable/reference/requirements-file-format/)
