# Solution

**k-fold cross-validation** evaluates a model on `k` different train/test partitions and reports the spread across folds, not a single lucky split. The MLOps discipline this task teaches is **tracking that structure in MLflow**: rather than one opaque number, each fold is logged as its own **nested run** under a single parent run, so the folds are independently visible and comparable in the UI while still rolling up under one experiment entry. The ML-methodology details are pre-wired for you — the splitter is already `StratifiedKFold` (so each fold preserves the class ratio on the imbalanced dataset) and the aggregate already reports both the mean and the standard deviation of each metric. Your work is to finish the per-fold metric computation and author the nested run per fold.

> As an MLOps engineer, you track a cross-validation experiment as nested MLflow runs—the ML-methodology details (stratified folds, mean/std spread) are handled for you. The data is synthetic; model quality is not the point.

#### Follow the steps below

##### 1. Confirm the starting state.
Open the **MLflow UI** button at the top of the lab — the `fraud-detection-cv` experiment is present and empty. From a VS Code terminal:
```
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:5000/
wc -l /root/code/fraud-detection/data/train.csv
```
A `200` confirms MLflow is reachable. The CSV has 201 lines (header + 200 rows).

##### 2. Inspect `cross_validate.py`.
Open `/root/code/fraud-detection/src/models/cross_validate.py` in the VS Code editor. Two TODO blocks inside the fold loop need work; the `StratifiedKFold` splitter and the mean/std `aggregate` dict are already in place:
- **TODO 1** — inside the fold loop, the `fold` dict is left as just `{"fold": fold_idx}`; the per-fold metric computation is unfinished. `preds` and `proba` are already computed just above it.
- **TODO 2** — each fold is not yet tracked. The `cv-parent` run wraps the loop, but there is no nested run per fold, so the individual folds never appear in the MLflow UI.

##### 3. TODO 1 — fill in the per-fold metrics.
Extend the `fold` dict **in place** so each fold records its three metrics — keep the existing `"fold": fold_idx` entry and add the three below it. `accuracy_score`, `f1_score`, and `roc_auc_score` are already imported; use `proba` (class-1 probabilities) for `roc_auc`:
```python
            fold = {
                "fold": fold_idx,
                "accuracy": round(accuracy_score(y_test, preds), 6),
                "f1": round(f1_score(y_test, preds), 6),
                "roc_auc": round(roc_auc_score(y_test, proba), 6),
            }
```
Leave the pre-wired `fold_results.append(fold)` line (at the end of the loop body) untouched — it records the fold for the aggregate report.

##### 4. TODO 2 — track each fold as a nested run.
In the TODO 2 gap (between the `fold` dict and the `fold_results.append(fold)` line), open a **nested** MLflow run for the fold and log its number and metrics. Passing `nested=True` makes it a child of the surrounding `cv-parent` run:
```python
            with mlflow.start_run(run_name=f"fold-{fold_idx}", nested=True):
                mlflow.log_param("fold", fold_idx)
                mlflow.log_metric("accuracy", fold["accuracy"])
                mlflow.log_metric("f1", fold["f1"])
                mlflow.log_metric("roc_auc", fold["roc_auc"])
```
Save the file.

##### 5. Run the evaluator.
```
python3 /root/code/fraud-detection/src/models/cross_validate.py
cat /root/code/fraud-detection/reports/cv_results.json
```
The JSON contains every `mean_*` and `std_*` key plus the five-element `folds` list.

##### 6. Verify in the MLflow UI.
Open the **MLflow UI** button → `fraud-detection-cv` experiment. The run list shows one parent (`cv-parent`) with five nested children (`fold-1` through `fold-5`) — expand the parent to see them grouped underneath. Click into each fold run to see its own `accuracy`, `f1`, and `roc_auc`; the parent carries the aggregate metrics that match the JSON report.

#### References
- MLflow nested runs (`start_run(nested=True)` — parent/child run tracking): https://mlflow.org/docs/latest/python_api/mlflow.html
- scikit-learn — cross-validation (k-fold, stratification, why it matters): https://scikit-learn.org/stable/modules/cross_validation.html
- `StratifiedKFold` (preserving class ratio across folds): https://scikit-learn.org/stable/modules/generated/sklearn.model_selection.StratifiedKFold.html
