# Solution

The MLflow **tracking server** is a long-running process that records experiment metadata (runs, parameters, metrics) in a *backend store* and saves run files (models, plots, datasets) in an *artifact store*. Training code points its tracking URI at this server to log runs, and the web UI reads from the same two stores. This task brings the server up with a SQLite backend store and a local artifact directory, then exposes it through the lab's browser proxy.

> As an MLOps engineer, you stand up the tracking server that captures every team's runs — the shared infrastructure behind experiment tracking — you are not training or evaluating any model here.

#### Follow the steps below

##### 1. Create the backend and artifact directories.
MLflow needs two filesystem locations: a backend directory for its SQLite database (metadata store) and an artifact directory for logged models, plots, and datasets. Keep both under `/root/code/` so they stay visible in the VS Code file explorer.
```
mkdir -p /root/code/mlflow-backend /root/code/mlflow-artifacts
```

##### 2. Start the MLflow tracking server in the background.
Launch the server with the SQLite backend, the local artifact directory, and the flags required for the lab's browser proxy. The `--allowed-hosts` and `--cors-allowed-origins` flags let the `5000-port-*.kk-lab-dev.kodekloud.com` proxy URL reach the API — without them, the browser returns a CORS error even though `curl` against `localhost` succeeds. The trailing `&` backgrounds the process so it outlives the terminal session.
```
mlflow server \
  --backend-store-uri sqlite:////root/code/mlflow-backend/mlflow.db \
  --default-artifact-root /root/code/mlflow-artifacts \
  --host 0.0.0.0 \
  --port 5000 \
  --allowed-hosts '*' \
  --cors-allowed-origins '*' &
```

##### 3. Verify that the server is reachable from the controlplane.
Once the server is up, `curl` returns the MLflow homepage HTML:
```
curl -s http://localhost:5000/ | head -c 200
```
A successful response begins with `<!doctype html>` and references the MLflow favicon.

##### 4. Confirm the UI in the browser.
Open the **MLflow UI** button at the top of the lab. The `Default` experiment is listed in the left-hand panel. The experiment is empty.

#### References
- MLflow `mlflow server` CLI reference: https://mlflow.org/docs/latest/cli.html
- MLflow tracking server (host, artifact root, proxy): https://mlflow.org/docs/latest/tracking/server.html
- MLflow backend stores (SQLite `--backend-store-uri`): https://mlflow.org/docs/latest/tracking/backend-stores.html
