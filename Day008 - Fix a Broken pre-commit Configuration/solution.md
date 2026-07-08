# Solution

`pre-commit` is a framework that runs small checks (hooks) automatically on `git commit`, with each hook declared in `.pre-commit-config.yaml` by its `repo`, pinned `rev`, and hook `id`. In this task you fix a broken config for the fraud-detection repo — correcting moved repository URLs, a deprecated and a malformed hook id, and a missing `rev:` pin — declare the five required hooks (`trailing-whitespace`, `end-of-file-fixer`, `check-yaml`, `ruff`, `black`), then `autoupdate` the pins and run the hooks across every tracked file.

> As an MLOps engineer, you wire code-quality checks into the commit workflow so every change is validated automatically before it lands — you are not writing model code here.

#### Follow the steps below

**About `pre-commit`:** `pre-commit` is a framework that runs small checks (hooks) automatically on `git commit`. Hooks are declared in `.pre-commit-config.yaml`: each entry names a `repo` (the git URL hosting the hooks), a `rev` (the pinned release tag of that repo), and the `hooks` to run from it (by `id`). `pre-commit install` writes the git hook so the checks run on every commit; `pre-commit run --all-files` runs them across the whole repo; and `pre-commit autoupdate` bumps each `rev` to its repository's latest release.

##### 1. Observe the current output.
Change into the project directory, install the git hook, and run all hooks against the tracked files so that the current problem is visible.
```
cd /root/code/fraud-detection
pre-commit install
pre-commit run --all-files
```
`pre-commit install` succeeds because it only writes the git hook script — it does not validate the rest of the configuration. `pre-commit run --all-files` then fails: the config references an outdated rev, a moved repository, a deprecated hook id, an unknown hook id written with an underscore, and a repository entry with no `rev:` pin.

##### 2. Fix the structural issues.
Replace the configuration with a version that corrects the hook ids and repository URLs. Any `rev:` value is acceptable at this stage — the next step will bump every pin to the latest release automatically.
```
cat > .pre-commit-config.yaml << 'YMLEOF'
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v2.3.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml

  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.1.0
    hooks:
      - id: ruff

  - repo: https://github.com/psf/black-pre-commit-mirror
    rev: v0.1.0
    hooks:
      - id: black
YMLEOF
```

##### 3. Bump every rev to the latest release.
`pre-commit autoupdate` queries each referenced repository and rewrites the `rev:` pins to match the latest tag. This is how teams keep hook versions fresh without looking them up by hand.
```
pre-commit autoupdate
```

##### 4. Run the hooks against every tracked file.
The first invocation downloads and caches each hook's environment; subsequent runs are much faster. The `trailing-whitespace` hook will correct the trailing spaces in `process.py` on its first pass.
```
pre-commit run --all-files
```
Because that hook *modified* a file, this first run reports `trailing-whitespace ... Failed` and exits non-zero — that is expected. Run it once more and, with `process.py` now clean, every hook passes:
```
pre-commit run --all-files
```

##### 5. Verify.
Confirm that the configuration parses cleanly and the git hook file is in place.
```
pre-commit validate-config
ls -l .git/hooks/pre-commit
```

---

**References:**
- [pre-commit — official documentation](https://pre-commit.com/)
