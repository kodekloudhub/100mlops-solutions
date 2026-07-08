# Solution

A model **bake-off** trains several candidate models on the same problem and picks a winner by a shared metric. When each candidate is logged as its own MLflow run — tagged with its model family and carrying its `f1_score` — selection becomes a *query*: `mlflow.search_runs(...)` returns the runs as a table you can sort. The orchestrator's job is to sort by the metric in the **right direction** (descending, so the top row is the best F1 — an ascending sort would hand back the *worst* candidate) and to record **which family won** (its `tags.candidate`), so downstream tooling promotes the right model. This task fixes an orchestrator that sorts the wrong way and omits the winning family.

> As an MLOps engineer, you run a model-selection bake-off—comparing candidates by querying MLflow and recording the winner—you are not deciding which algorithm is genuinely best. The data is synthetic and `f1_score` is just a selection key.

#### Follow the steps below

##### 1. Confirm the starting state.
Open the **MLflow UI** button at the top of the lab — the `bakeoff` experiment is present and empty. From a VS Code terminal:
```
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:5000/
ls /root/code/fraud-detection/src/models/
```
A `200` confirms MLflow is reachable. The three trainer scripts (`train_rf.py`, `train_gb.py`, `train_lr.py`) and the orchestrator (`bakeoff.py`) are already staged.

##### 2. Run the three trainers.
Each script trains its named estimator under 3-fold stratified CV and logs one run to the `bakeoff` experiment with the candidate tag, hyperparameters, and mean `f1_score`.
```
python3 /root/code/fraud-detection/src/models/train_rf.py
python3 /root/code/fraud-detection/src/models/train_gb.py
python3 /root/code/fraud-detection/src/models/train_lr.py
```
Each script prints its own `f1_score`. Open the **MLflow UI** → `bakeoff` experiment and confirm three runs are visible, tagged `random_forest`, `gradient_boosting`, and `logistic_regression`.

##### 3. Run the draft orchestrator and observe the gaps.
```
python3 /root/code/fraud-detection/src/models/bakeoff.py
cat /root/code/fraud-detection/reports/winner.json
```
The report is written, but two things are wrong. Compare its `f1_score` to the three per-candidate numbers printed in Step 2 — the saved value is the lowest of the three, not the highest. And the JSON has no `model_type` key at all, so there is no way to tell which model family won.

##### 4. Inspect `bakeoff.py`.
Open `/root/code/fraud-detection/src/models/bakeoff.py` in the VS Code editor. Two problems are visible:
- `search_runs(..., order_by=["metrics.f1_score ASC"], ...)` — on an ascending sort, `runs.iloc[0]` is the *worst* candidate. The orchestrator needs to sort descending so the top row is the best.
- The `report` dict only contains `run_id` and `f1_score`. The winning run's `tags.candidate` is never read, so the report cannot say which model family won.

##### 5. Fix the sort direction.
Change the `order_by` argument to descending:
```python
    runs = mlflow.search_runs(
        experiment_ids=[exp.experiment_id],
        order_by=["metrics.f1_score DESC"],
        max_results=10,
    )
```

##### 6. Add `model_type` to the report.
Read the winner's candidate tag from the row and put it in the report as `model_type`. `search_runs` surfaces the candidate tag under the column `tags.candidate`:
```python
    winner = runs.iloc[0]
    report = {
        "model_type": winner["tags.candidate"],
        "run_id":     winner["run_id"],
        "f1_score":   float(winner["metrics.f1_score"]),
    }
```
Save the file.

##### 7. Re-run the orchestrator.
```
python3 /root/code/fraud-detection/src/models/bakeoff.py
cat /root/code/fraud-detection/reports/winner.json
```
The JSON now carries `model_type`, `run_id`, and `f1_score`. The `f1_score` matches the highest of the three numbers printed by the trainer scripts.

##### 8. Verify in the MLflow UI.
Open the **MLflow UI** button → `bakeoff` experiment. Select every run in the list and click **Compare** — the Compare view lines all three candidates up side by side, and the candidate with the tallest `f1_score` bar matches the `model_type` saved to `winner.json`.

#### References
- `mlflow.search_runs` (returns runs as a sortable table; `order_by`, `tags.*`/`metrics.*` columns): https://mlflow.org/docs/latest/python_api/mlflow.html
- MLflow search syntax (ordering and filtering runs): https://mlflow.org/docs/latest/search-runs.html
- MLflow Tracking UI — comparing runs side by side (the Compare view): https://mlflow.org/docs/latest/tracking.html
