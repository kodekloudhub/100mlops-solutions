# Solution

`ruff` is a fast Python linter and `black` is an opinionated formatter; both read their settings from `pyproject.toml` and are run in CI so every pull request meets the same code-quality bar. In this task you fix the fraud-detection project's configuration — moving `ruff`'s rule selection under `[tool.ruff.lint]` (the v0.1+ schema), setting both tools to a line length of `120`, and adding the `W` and `I` rules — then apply each tool's autofix so `ruff check src/` and `black --check src/` both exit cleanly.

> As an MLOps engineer, you make code-quality checks automatic and uniform so the team stops arguing about formatting in review — the ML logic is incidental.

#### Follow the steps below

**About `ruff` and `black`:** `ruff` is a fast Python *linter* — it flags code-quality issues (unused imports, undefined names, unsorted imports, style problems) according to the rule families you enable: `E`/`W` (pycodestyle errors/warnings), `F` (pyflakes), `I` (import sorting). `black` is an opinionated *formatter* — it rewrites code to one canonical style (for example, normalising single-quoted strings to double quotes). Both read their settings from `pyproject.toml`: `ruff` under `[tool.ruff]`, with lint rules under `[tool.ruff.lint]` since v0.1, and `black` under `[tool.black]`. Teams run both in CI so every pull request meets the same bar.

##### 1. Observe the current failures.
Change into the project directory and run both tools so that the existing problems are visible.
```
cd /root/code/fraud-detection
ruff check src/
black --check src/
```
Ruff reports both a configuration schema warning and a lint violation in one of the source files. Black reports that at least one file would be reformatted.

##### 2. Inspect the existing configuration.
Review `pyproject.toml` and compare each field against the requirements in the task.
```
cat pyproject.toml
```
The file puts `select` directly under `[tool.ruff]` (the legacy schema), the ruff line length is `88`, the ruff select list is missing `W` and `I`, and the black line length is `100`.

##### 3. Rewrite `pyproject.toml`.
Replace the file with a configuration that satisfies every requirement.
```
cat > pyproject.toml << 'TOMLEOF'
[project]
name = "fraud-detection"
version = "0.1.0"

[tool.ruff]
line-length = 120

[tool.ruff.lint]
select = ["E", "F", "W", "I"]

[tool.black]
line-length = 120
TOMLEOF
```

##### 4. Fix the remaining source issues.
With the configuration corrected, let each tool apply its autofix. `ruff check --fix src/` removes the unused `os` import and sorts the import block; `black src/` reformats the code to its canonical style (here it normalises a single-quoted string to double quotes).
```
ruff check --fix src/
black src/
```

##### 5. Confirm both tools pass.
```
ruff check src/
black --check src/
```
Both commands must exit with status `0`.

##### 6. Verify.
```
grep -E 'line-length|select' pyproject.toml
```

---

**References:**
- [Ruff — Configuring Ruff](https://docs.astral.sh/ruff/configuration/)
- [Black — configuration basics](https://black.readthedocs.io/en/stable/usage_and_configuration/the_basics.html)
