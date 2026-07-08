# Solution

Large datasets do not belong in Git — they bloat the repository and Git is not built for big binary files. `dvc add <file>` hands the file to DVC: it copies the data into DVC's cache, writes a tiny `<file>.dvc` *pointer* (recording the file's hash and path), and adds the file to a `.gitignore` so Git stops seeing the data itself. You then commit the small `.dvc` pointer and `.gitignore` to Git — Git versions the *pointer*, DVC versions the *data*. Moving an already-committed file to DVC starts with `git rm --cached`, which unstages it from Git while keeping it on disk.

> As an MLOps engineer, you move a dataset from Git to DVC so code and data are versioned by the right tool — you are not analysing the data; the dataset is synthetic.

#### Follow the steps below

##### 1. Inspect the current tracking situation.
Move into the project and confirm that the dataset is currently tracked by Git rather than by DVC.
```
cd /root/code/fraud-detection
git log --oneline
git ls-files data/raw/
```
The most recent commit added `data/raw/transactions.csv` directly to Git, and `git ls-files` confirms Git is currently tracking the dataset.

##### 2. Stop Git from tracking the dataset.
`git rm --cached` removes the file from the Git index without deleting it from disk. The working-tree copy stays in place so DVC can pick it up in the next step.
```
git rm --cached data/raw/transactions.csv
```

##### 3. Track the dataset with DVC.
`dvc add` records the dataset's hash in a small `.dvc` pointer file and writes a per-directory `.gitignore` so Git no longer sees the dataset itself.
```
dvc add data/raw/transactions.csv
```
Two new files appear: `data/raw/transactions.csv.dvc` and `data/raw/.gitignore`.

##### 4. Commit the cleanup.
A single commit captures three coordinated changes — the removal of the dataset from Git, the new `.dvc` pointer, and the new `.gitignore` — leaving the repository in a clean, coherent state.
```
git add data/raw/transactions.csv.dvc data/raw/.gitignore
git commit -m "Track transactions dataset with DVC"
```

##### 5. Verify.
Confirm Git no longer tracks the CSV, DVC owns it via the pointer, and the file is still present on disk.
```
git ls-files data/raw/
ls -la data/raw/
cat data/raw/transactions.csv.dvc
```
The DVC extension's **DVC TRACKED** section in the EXPLORER panel now lists `transactions.csv` as a tracked file, confirming the move from Git to DVC.

---

**References:**
- [DVC — `dvc add`](https://dvc.org/doc/command-reference/add)
- [DVC — Data Versioning](https://dvc.org/doc/start/data-management/data-versioning)
