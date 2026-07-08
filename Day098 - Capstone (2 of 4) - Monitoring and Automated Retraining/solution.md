# Solution

**The platform.** xFusionCorp runs `fraud-detector` as a small production ML system: models are trained and served behind the `production` registry alias, an automated loop retrains on drift, the server is deployed by GitOps, and the running service is observed with Prometheus + Grafana. **This task builds the automated retraining loop** — the part that keeps the served model current as the data moves.

You compose the closed loop: a drift monitor quantifies the shift, a gate decides whether retraining is warranted, and promotion moves the serving alias to the new version. You complete the two TODOs in `retrain_if_drift.py` — the drift gate and the register + promote step — then run it and confirm version 2 is live under the `production` alias. Because the serving layer resolves that alias, promotion goes live without a redeploy.

> As an MLOps engineer, you wire detect, retrain, and promote into one script so the system retrains itself when the data drifts instead of when someone notices — you are not tuning the model; the model and data are synthetic.

#### Follow the steps below

##### 1. Confirm the pre-staged environment.
From a VS Code terminal:
```
curl -s -o /dev/null -w 'mlflow=%{http_code}\n' http://localhost:5000/
curl -s -o /dev/null -w 'reports=%{http_code}\n' http://localhost:8086/
python3 -c "
import mlflow
mlflow.set_tracking_uri('http://localhost:5000')
c = mlflow.tracking.MlflowClient()
v = c.get_model_version_by_alias('fraud-detector', 'production')
print(f'fraud-detector @production = v{v.version}')
"
```
MLflow and the reports server both answer `200`, and the alias resolves to `v1`. `/root/code/reports/` has no `drift.html` yet — you generate it by running the loop.

##### 2. Read the scaffolded loop.
Open `/root/code/retrain_if_drift.py`. The plumbing is written — it runs `drift.py`, reads `drift-summary.json` into a `drifted` boolean, runs `retrain.py`, and resolves the new run id. Two TODOs remain: the drift gate (TODO 1) and the promote step (TODO 2).

##### 3. Author TODO 1 — gate retraining on drift.
Only retrain when the data has actually drifted:
```python
if not drifted:
    print("[loop] no drift; model is still current -- skipping retrain")
    sys.exit(0)
```

##### 4. Author TODO 2 — register + promote in code.
After `retrain.py` runs and `run_id` is resolved, register the new run and move the alias:
```python
mv = mlflow.register_model(f"runs:/{run_id}/model", MODEL_NAME)
client.set_registered_model_alias(MODEL_NAME, ALIAS, mv.version)
print(f"[loop] promoted {MODEL_NAME} v{mv.version} -> @{ALIAS}")
```
`set_registered_model_alias` reassigns `production` exclusively — it moves off v1 onto the new version. Save.

##### 5. Run the loop.
```
cd /root/code
python3 retrain_if_drift.py
```
It prints `dataset_drift=True`, retrains, and promotes `fraud-detector v2 -> @production`. Click the **Drift Report** button → **drift.html**: Evidently's per-column cards show the `reference` vs `current` shift, and the summary card reports dataset drift `True`.

##### 6. Verify the promotion.
In the **MLflow UI → Models → fraud-detector**, two versions are listed and `@production` now sits on Version 2. From the terminal:
```
python3 -c "
import mlflow
mlflow.set_tracking_uri('http://localhost:5000')
c = mlflow.tracking.MlflowClient()
v = c.get_model_version_by_alias('fraud-detector', 'production')
print(f'fraud-detector @production = v{v.version}')
"
```
Output: `fraud-detector @production = v2`.

#### References

- MLflow — Model Registry (registering runs, versions, and aliases via `MlflowClient`): https://mlflow.org/docs/latest/ml/model-registry/
- Evidently — data drift report (`ValueDrift` + `DriftedColumnsCount` metrics): https://docs.evidentlyai.com/
