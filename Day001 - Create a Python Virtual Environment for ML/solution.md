# Solution

A virtual environment is an isolated, per-project Python sandbox created with `python3 -m venv`, so each project can pin its own package versions without colliding with the system Python or other projects. In this task you create an `ml-env` environment under `/root/code/`, install the core ML libraries (`numpy`, `pandas`, `scikit-learn`, `matplotlib`) into it, and freeze the exact versions to a `requirements.txt` so the setup is reproducible for the whole team.

> As an MLOps engineer, you standardize the project's dependency setup so every teammate's environment is byte-for-byte reproducible — you are not writing model code here.

#### Follow the steps below

##### 1. Create the virtual environment.
A virtual environment is a private, isolated Python sandbox for a single project. Anything installed into it stays inside it, so different projects can rely on different package versions without interfering with one another.
```
python3 -m venv /root/code/ml-env
```

##### 2. Activate the virtual environment.
Activating the environment tells the shell that any subsequent `python` or `pip` command must resolve to the binaries inside the sandbox. The prompt will display `(ml-env)` once the environment is active.
```
source /root/code/ml-env/bin/activate
```

##### 3. Install the required packages.
`numpy` and `pandas` handle data manipulation, `scikit-learn` provides the modelling primitives, and `matplotlib` produces plots. Because the virtual environment is active, these packages are installed into `ml-env/` rather than into the system Python.
```
pip install numpy pandas scikit-learn matplotlib
```

##### 4. Generate the requirements.txt file.
`pip freeze` writes every installed package to standard output with its exact version. Anyone — or any CI pipeline — can then run `pip install -r requirements.txt` to reproduce the same environment.
```
pip freeze > /root/code/requirements.txt
```

##### 5. Verify the installation.
`pip list` shows every installed package. Inspect the `requirements.txt` file to confirm the pinned versions.
```
pip list
cat /root/code/requirements.txt
```

---

**References:**
- [Python `venv` — Creation of virtual environments](https://docs.python.org/3/library/venv.html)
- [`pip freeze` — pip documentation](https://pip.pypa.io/en/stable/cli/pip_freeze/)
