# Solution

Model **evaluation** produces the evidence a release decision rests on: a fixed set of metrics computed on a held-out test set, plus a confusion matrix, written to a known location and logged to MLflow so every candidate model is judged the same way. A subtlety worth internalising — different metrics need different inputs: accuracy, precision, recall, and F1 are computed from the model's hard class predictions, while **AUC-ROC needs the model's class-1 probabilities** (`predict_proba(...)[:, 1]`), not the 0/1 labels. This task repairs a draft evaluator so its report carries all five required metrics under the expected key names and lands in the project's `reports/` directory.

> As an MLOps engineer, you produce a standardized, correctly-located, MLflow-logged metrics report—you are not choosing metrics or judging whether the model is good. The data and model are synthetic.

#### Follow the steps below

##### 1. Confirm the starting state.
Open the **MLflow UI** button at the top of the lab — the `fraud-detection-eval` experiment is present and empty. From a VS Code terminal:
```
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:5000/
ls -l /root/code/fraud-detection/models/model.pkl /root/code/fraud-detection/data/test.csv
```
A `200` confirms MLflow is reachable. Both `model.pkl` and `test.csv` already exist — the startup pre-trained the model.

##### 2. Run the draft evaluator and observe the output.
```
python3 /root/code/fraud-detection/src/models/evaluate.py
ls -l /root/code/fraud-detection/reports/
```
The confusion-matrix PNG lands where the release checklist wants it. The metrics JSON is nowhere to be found under `reports/` — the script wrote it to `/tmp/metrics.json` instead. Opening that file reveals only two keys (`accuracy`, `f1`).

##### 3. Inspect `evaluate.py`.
Open `/root/code/fraud-detection/src/models/evaluate.py` in the VS Code editor. Three problems are visible:
- `METRICS_JSON = "/tmp/metrics.json"` — the destination is outside the project tree.
- The `metrics` dict inside `main()` computes only `accuracy` and `f1`. Three metrics the release checklist requires (`precision`, `recall`, `auc_roc`) are never computed.
- The f1 metric is stored under the key `"f1"` — downstream tooling expects `"f1_score"`.

##### 4. Fix the destination path.
Change the constant so the metrics JSON lands next to the confusion-matrix image:
```python
METRICS_JSON = os.path.join(REPORTS_DIR, "metrics.json")
```

##### 5. Fix the metrics block.
The three missing metric functions (`precision_score`, `recall_score`, `roc_auc_score`) are already imported at the top of the file — no new imports are needed. Populate every key the release checklist expects. Use `predict_proba(...)[:, 1]` for `auc_roc` because the metric wants class-1 probabilities, not hard predictions:
```python
metrics = {
    "accuracy": round(accuracy_score(y, preds), 6),
    "precision": round(precision_score(y, preds), 6),
    "recall": round(recall_score(y, preds), 6),
    "f1_score": round(f1_score(y, preds), 6),
    "auc_roc": round(roc_auc_score(y, proba), 6),
}
```
Save the file.

##### 6. Re-run the evaluator.
```
python3 /root/code/fraud-detection/src/models/evaluate.py
```
The output now lists all five metric keys, and both files are written inside `/root/code/fraud-detection/reports/`:
```
ls -l /root/code/fraud-detection/reports/
cat /root/code/fraud-detection/reports/metrics.json
```

##### 7. Verify in the MLflow UI.
Open the **MLflow UI** button → `fraud-detection-eval` experiment → click the newest run. Three panels confirm the result:
- **Metrics** lists `accuracy`, `precision`, `recall`, `f1_score`, `auc_roc`.
- **Artifacts** contains `confusion_matrix.png` and `metrics.json`.
- The confusion matrix renders inline when the PNG is selected.

#### References
- scikit-learn — classification metrics (accuracy, precision, recall, F1, ROC-AUC): https://scikit-learn.org/stable/modules/model_evaluation.html
- `roc_auc_score` (why it takes probabilities, not hard predictions): https://scikit-learn.org/stable/modules/generated/sklearn.metrics.roc_auc_score.html
- MLflow logging (`log_metric` / `log_artifact`): https://mlflow.org/docs/latest/python_api/mlflow.html
