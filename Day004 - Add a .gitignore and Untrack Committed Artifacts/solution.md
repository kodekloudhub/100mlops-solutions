# Solution

A `.gitignore` tells Git which paths to leave untracked, but it has no effect on files already committed — those must be removed from the index with `git rm --cached` while staying on disk. In this task you add a `.gitignore` covering the standard Python/ML artifacts (`__pycache__/`, `*.pyc`, `venv/`, `.ipynb_checkpoints/`, `*.pkl`, `.env`) to the fraud-detection repo, then untrack the ones that were committed before it existed and commit the cleanup, leaving only the real sources tracked.

> As an MLOps engineer, you keep the repository clean of caches, environments, and secrets so only real sources are versioned — you are not writing model code here.

#### Follow the steps below

**About `.gitignore` and untracking:** `.gitignore` tells Git which paths to leave *untracked*, but it has no effect on files Git already tracks. To stop tracking a file that was committed before it was ignored, remove it from the index with `git rm --cached` (which deletes it from Git but leaves the working copy on disk) and commit that change.

##### 1. Inspect what is currently tracked.
Change into the repository and list the tracked files. The listing includes the artifacts that should not be there.
```
cd /root/code/fraud-detection
git ls-files
```
Alongside the real sources you will see `src/fraud_detection/__pycache__/…`, `models/fraud_model.pkl`, `venv/…`, `.ipynb_checkpoints/…`, and `.env`.

##### 2. Create the `.gitignore`.
Write the ignore patterns for the standard Python / ML artifacts at the repository root.
```
cat > .gitignore << 'EOF'
__pycache__/
*.pyc
venv/
.ipynb_checkpoints/
*.pkl
.env
EOF
```

##### 3. Untrack the already-committed artifacts.
`git rm -r --cached` removes the paths from the index only; the files stay in the working tree.
```
git rm -r --cached src/fraud_detection/__pycache__ models/fraud_model.pkl venv .ipynb_checkpoints .env
```

##### 4. Commit the cleanup.
```
git add .gitignore
git commit -m "Add .gitignore and untrack committed artifacts"
```

##### 5. Verify.
`git ls-files` now lists only the sources plus `.gitignore`; the artifacts are gone from Git but still present on disk.
```
git ls-files
ls -a && ls -a models
```

---

**References:**
- [Git — `gitignore` documentation](https://git-scm.com/docs/gitignore)
- [`git rm --cached` — removing files from the index](https://git-scm.com/docs/git-rm)
- [GitHub — Python `.gitignore` template](https://github.com/github/gitignore/blob/main/Python.gitignore)
