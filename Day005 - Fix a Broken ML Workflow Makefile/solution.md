# Solution

`make` is a build-automation tool that runs named **targets** (each a group of shell commands) and resolves the dependencies between them, with two hard rules: recipe lines must be indented with a real tab, and non-file targets belong under `.PHONY`. In this task you repair a broken `Makefile` for the ML pipeline — fixing a space-indented recipe, adding the missing `.PHONY` declaration, wiring `data` into the `all` chain, and completing the `clean` target — so `make all` runs `setup`, `data`, `train`, and `test` end to end without error.

> As an MLOps engineer, you standardize the project's routine tasks behind one-word commands so the whole pipeline runs the same way for everyone — you are not writing model code here.

#### Follow the steps below

**About `make`:** `make` is a build-automation tool that runs **targets** — named groups of shell commands called **recipes** — and resolves the dependencies between them. A `Makefile` declares each target as `name:` followed by its recipe lines, and `make <target>` runs that target after any prerequisites listed after the colon. Two rules matter here: every recipe line **must be indented with a real tab** (Make rejects spaces), and any target that does not create a file of its own name should be listed under **`.PHONY`** so Make always runs it instead of skipping it.

##### 1. Observe the failure.
Change into the project directory and attempt to run the full pipeline.
```
cd /root/code/fraud-detection
make all
```
Make reports a parse error immediately because one of the recipes is not tab-indented.

##### 2. Inspect the current Makefile.
Review each target and compare it against the behaviour required by the task.
```
cat -A Makefile
```
`cat -A` renders tabs as `^I` and line endings as `$`, which makes whitespace mistakes visible. Four issues are present: the file has no `.PHONY` declaration, the `data` recipe is indented with spaces instead of a tab, the `all` target does not include `data` in its dependency chain, and the `clean` target neither recurses nor clears every required path.

##### 3. Rewrite the Makefile.
Build the corrected file with `printf`. The `\t` escape produces a real tab character, which removes any risk of accidentally indenting with spaces.
```
{
  printf '.PHONY: setup data train test clean all\n\n'
  printf 'setup:\n\tpython3 -m venv mlops-venv && mlops-venv/bin/pip install -r requirements.txt\n\n'
  printf 'data:\n\tpython3 src/data/process_data.py\n\n'
  printf 'train:\n\tpython3 src/models/train.py\n\n'
  printf 'test:\n\tpytest tests/\n\n'
  printf 'clean:\n\tfind . -type d -name __pycache__ -exec rm -rf {} +\n\trm -rf .pytest_cache\n\trm -rf models/*\n\n'
  printf 'all: setup data train test\n'
} > Makefile
```

##### 4. Confirm the pipeline runs cleanly.
A dry run verifies the dependency graph; a full run of `clean` verifies the expanded recipe.
```
make -n all
make clean
```

##### 5. Verify.
```
grep -E '^\.PHONY' Makefile
grep -E '^all:' Makefile
cat -A Makefile | head -20
```

---

**References:**
- [Makefile Tutorial — targets, `.PHONY`, and the tab-indentation rule](https://makefiletutorial.com/)
