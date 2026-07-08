# Solution

This capstone walks one model through the back half of the MLOps lifecycle on a production-style MLflow stack (PostgreSQL metadata + SeaweedFS artifacts). The candidate runs already exist, so the work is **promotion → serving → monitoring**: pick the best run by `f1_score`, register it and move the `champion` alias onto that version, stand it up behind a real inference endpoint with `mlflow models serve`, and add a health probe so an external scheduler can tell whether the endpoint is alive. The `champion` alias is the seam between the registry and serving — `mlflow models serve -m "models:/<name>@champion"` always serves whatever version the alias currently points at, so promoting a new model later is just an alias move.

> As an MLOps engineer, you carry the champion through promotion, serving, and health monitoring so the endpoint the alias points at is live and observable — you are not choosing the model on merit; the candidate runs' metrics are synthetic.

#### Follow the steps below

##### 1. Confirm the starting state.
PostgreSQL, SeaweedFS, the MLflow tracking server, and the three candidate runs in `fraud-detection-v2` are already in place. Confirm their health from the terminal:
```
docker exec mlflow-db pg_isready -U mlflow -d mlflow
curl -s -o /dev/null -w 'seaweed=%{http_code} ' http://localhost:8333/status
curl -s -o /dev/null -w 'mlflow=%{http_code}\n' http://localhost:5000/
```
Open the **MLflow UI** button → `Model training → Experiments → fraud-detection-v2` — the dashboard lists `baseline`, `improved`, and `regression`.

##### 2. Identify the top candidate.
In the **MLflow UI**, multi-select the three runs in `fraud-detection-v2` and click **Compare** to confirm the `improved` run (`f1_score = 0.92`) as the top candidate. The same conclusion is reachable from a terminal query:
```
python3 -c "
import mlflow
mlflow.set_tracking_uri('http://localhost:5000')
exp = mlflow.get_experiment_by_name('fraud-detection-v2')
runs = mlflow.search_runs([exp.experiment_id], order_by=['metrics.f1_score DESC'])
print(runs[['run_id','tags.mlflow.runName','metrics.f1_score']].head())
"
```

##### 3. Register the top run and assign the champion alias.
Either path works: the MLflow UI (per-experiment **Models** tab → click the row sourced from the top run → **Register model** → Create New Model → `fraud-detector-v2`; then **Model registry** → `fraud-detector-v2` → Version 1 → Aliases → `champion`) or the SDK. Registration is a metadata-only operation against the tracking server — no S3 credentials are needed. The SDK path is scriptable and lands the alias in a single shell block:
```
python3 -c "
import mlflow
from mlflow import MlflowClient
mlflow.set_tracking_uri('http://localhost:5000')
client = MlflowClient()
exp = client.get_experiment_by_name('fraud-detection-v2')
runs = mlflow.search_runs([exp.experiment_id], order_by=['metrics.f1_score DESC'])
best_run = runs.iloc[0]['run_id']
result = mlflow.register_model(f'runs:/{best_run}/model', 'fraud-detector-v2')
client.set_registered_model_alias('fraud-detector-v2', 'champion', result.version)
print(f'registered v{result.version} as champion')
"
```

##### 4. Serve the champion on port 5001.
The serving process resolves the `models:/` alias against the tracking server, and the tracking server proxies the model download from SeaweedFS itself — so the only variable the serving shell needs is `MLFLOW_TRACKING_URI`. Background the process with `nohup ... &` so the terminal stays free, then poll `/health` until the Flask app is up:
```
export MLFLOW_TRACKING_URI=http://localhost:5000

nohup mlflow models serve \
  -m "models:/fraud-detector-v2@champion" \
  --host 0.0.0.0 --port 5001 \
  --env-manager=local \
  > /tmp/serve.log 2>&1 &

for i in $(seq 1 30); do
  curl -sf -o /dev/null http://localhost:5001/health && break
  sleep 1
done
```

##### 5. Confirm the endpoint.
```
curl -s -o /dev/null -w 'health=%{http_code}\n' http://localhost:5001/health
curl -s -X POST http://localhost:5001/invocations \
  -H 'Content-Type: application/json' \
  -d '{"dataframe_split":{"columns":["a","b"],"data":[[0.1,0.5],[0.7,0.2]]}}'
```
`/health` returns `200`. `/invocations` returns a JSON body containing `predictions`.

##### 6. Author the monitoring script.
`/root/code/monitor.sh` is a one-shot health probe — hit `/health` on the served model and propagate the result via the script's **exit code**. Scheduled runners (`cron`, systemd timers, Kubernetes liveness probes) consume exactly this shape of signal: exit `0` means healthy, non-zero means unhealthy. Create the file in the VS Code editor (`/root/code/monitor.sh`) — authoring it in the editor avoids the paste issues a multi-line terminal heredoc can hit:
```bash
#!/usr/bin/env bash
set -u
if curl -sf -o /dev/null http://localhost:5001/health; then
  echo "healthy"
  exit 0
fi
echo "unhealthy"
exit 1
```
`curl -sf` returns a non-zero status on any HTTP error, so the `if` cleanly maps a healthy `/health` to exit `0` and anything else to exit `1`. Save it, then make it executable:
```
chmod +x /root/code/monitor.sh
```

##### 7. Verify.
```
/root/code/monitor.sh
echo "exit=$?"
```
The script prints `healthy` and exits with status `0`. Open **MLflow UI** → **Model registry** → **fraud-detector-v2** — a single version with the `champion` alias is shown. Open **SeaweedFS Filer** → navigate to `/buckets/mlflow-artifacts/` — the run's model directory is present.

#### References
- MLflow Model Registry — registered models, versions, aliases: https://mlflow.org/docs/latest/model-registry.html
- Deploy an MLflow model locally — `mlflow models serve`, `/health`, `/invocations`: https://mlflow.org/docs/latest/deployment/deploy-model-locally.html
- MLflow CLI (`mlflow models serve` flags): https://mlflow.org/docs/latest/cli.html
