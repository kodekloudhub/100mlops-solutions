# Solution

Git versions your code, but datasets and model files are too large to live in Git. DVC fills the gap by keeping the actual data and model artefacts in its own cache and committing only small **pointer files** to Git — the `.dvc` file for data tracked with `dvc add`, and `dvc.lock` for pipeline outputs such as the trained model. Because those pointers are ordinary Git objects, a Git branch or tag captures an exact data + model state alongside the code that produced it. Switching branches changes the pointers, and `dvc checkout` then materialises the matching data and model from the cache — so `main` and `v2-improved` can hold entirely different datasets and models, and you can roll between them cleanly.

> As an MLOps engineer, you version data and models alongside code on Git branches and tags so any state is reproducible from a single commit — you are not comparing model quality; the datasets are synthetic.

#### Follow the steps below

##### 1. Move into the project.
```
cd /root/code/fraud-detection
```

##### 2. Tag the current state as `v1.0`.
A Git tag is a permanent bookmark on the current commit. Because the dataset pointer (`transactions.csv.dvc`) and the model pointer (`dvc.lock`) are both in Git, tagging the commit locks in the dataset *and* model hashes — code, data, and model stay together.
```
git tag v1.0
```

##### 3. Create the `v2-improved` branch.
A new branch lets the team experiment with an updated dataset without disturbing the main branch. If v2 turns out to be worse, the branch can simply be discarded.
```
git checkout -b v2-improved
```

##### 4. Replace the tracked dataset with the v2 file.
The improved dataset has already been pre-staged at `data/raw/transactions_v2.csv`. Copy its contents over the tracked file so DVC sees a new hash for the same path.
```
cp data/raw/transactions_v2.csv data/raw/transactions.csv
```

##### 5. Re-track the updated dataset and re-run the pipeline.
`dvc add` recomputes the hash and updates the `.dvc` pointer; `dvc repro` then walks the pipeline and re-runs every stage that depends on the changed data — including `train`, so a new `models/model.pkl` is produced from the v2 data and its hash is recorded in `dvc.lock`.
```
dvc add data/raw/transactions.csv
dvc repro
```

##### 6. Commit the v2 changes.
This commit captures the v2 state — new dataset pointer, new pipeline outputs, and the new model recorded in `dvc.lock`.
```
git add .
git commit -m "Update dataset to v2 and retrain model"
```

##### 7. Switch back to main and restore the v1 dataset and model.
`git checkout main` flips the code and pointer files back to v1; `dvc checkout` reads those pointers and restores both the original CSV and the original `models/model.pkl` from DVC's local cache.
```
git checkout main
dvc checkout
```

##### 8. Verify.
The restored CSV must be the v1 baseline (smaller, original content), the v1 model must be back on disk, and `dvc status` must report no changes.
```
wc -l data/raw/transactions.csv
ls -l models/model.pkl
dvc status
git tag -l
git branch
```

> The DVC extension's **DVC TRACKED** section in the EXPLORER panel reflects the tracked dataset and model for the branch you currently have checked out. To compare the exact hashes recorded on each branch, use `git show v1.0:dvc.lock` / `git show v2-improved:dvc.lock` (or `dvc status`).

---

**References:**
- [DVC — Data Versioning](https://dvc.org/doc/use-cases/versioning-data-and-models)
- [DVC — `dvc checkout`](https://dvc.org/doc/command-reference/checkout)
- [DVC — `dvc add`](https://dvc.org/doc/command-reference/add)
