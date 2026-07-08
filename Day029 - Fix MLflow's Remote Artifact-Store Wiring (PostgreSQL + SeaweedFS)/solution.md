# Solution

A production MLflow deployment splits storage in two: a **backend store** for run metadata (here PostgreSQL) and an **artifact store** for files and models (here SeaweedFS, an S3-compatible object store). When the server runs with `--artifacts-destination s3://…`, it writes artifacts through its own **boto3** S3 client — and boto3, given only AWS credentials, talks to the real AWS endpoint (`s3.amazonaws.com`) unless `MLFLOW_S3_ENDPOINT_URL` points it at the local store. That single missing variable is why metadata (PostgreSQL) succeeds while artifact uploads (S3) fail. This task diagnoses that split-failure and repairs the server's S3 endpoint wiring.

> As an MLOps engineer, you wire the shared tracking server to its production backend and S3-compatible artifact store so runs persist to real infrastructure — you are not training or evaluating any model; the smoke-test model is synthetic.

#### Follow the steps below

##### 1. Confirm the three backing services are up.
From a VS Code terminal:
```
docker exec mlflow-db pg_isready -U mlflow -d mlflow
curl -s -o /dev/null -w 'seaweed=%{http_code}\n' http://localhost:8333/status
curl -s -o /dev/null -w 'mlflow=%{http_code}\n' http://localhost:5000/
AWS_ACCESS_KEY_ID=weedadmin AWS_SECRET_ACCESS_KEY=weedadmin123 \
  aws --endpoint-url=http://localhost:8333 s3api list-buckets \
  --query 'Buckets[].Name' --output text | grep mlflow-artifacts
```
PostgreSQL replies `accepting connections`; SeaweedFS answers `200`; MLflow answers `200`; the bucket-list call prints `mlflow-artifacts`.

##### 2. Reproduce the failure.
Run the pre-staged smoke-test:
```
python3 /root/code/log_test_run.py
```
The run appears in the MLflow UI (the metadata write to PostgreSQL succeeds), but the script **stalls** before printing `test-remote run logged successfully`. The MLflow server's boto3 client is trying to reach the real AWS endpoint (`s3.amazonaws.com`) to upload the model artefact; with no route to AWS from the lab, boto3 retries and the call hangs rather than returning promptly. Press `Ctrl+C` to interrupt it. (Where AWS *is* reachable you would instead get a fast traceback ending in `botocore.exceptions.ClientError: An error occurred (InvalidAccessKeyId) … PutObject` — either way the artefact upload never reaches SeaweedFS.) Open the **SeaweedFS Filer** → navigate to `/buckets/mlflow-artifacts/` — the bucket is still empty. The MLflow server is trying to upload to AWS S3 instead of the local SeaweedFS.

##### 3. Inspect the MLflow startup script.
Open `/root/code/start-mlflow.sh` in the VS Code editor:
```bash
export AWS_ACCESS_KEY_ID=weedadmin
export AWS_SECRET_ACCESS_KEY=weedadmin123

exec mlflow server \
  --backend-store-uri postgresql://mlflow:mlflow123@localhost:5432/mlflow \
  --artifacts-destination s3://mlflow-artifacts \
  ...
```
The script exports the S3 access keys but never tells boto3 where the S3 endpoint actually is. With no `MLFLOW_S3_ENDPOINT_URL` set, the MLflow server's boto3 client defaults to `s3.amazonaws.com`, which the lab's `weedadmin` credentials cannot authenticate against.

##### 4. Add the missing endpoint export.
Edit `/root/code/start-mlflow.sh` and add this line alongside the existing access-key exports:
```bash
export MLFLOW_S3_ENDPOINT_URL=http://localhost:8333
```
Save.

##### 5. Restart the MLflow server.
```
bash /root/code/restart-mlflow.sh
for i in $(seq 1 30); do
  curl -sf -o /dev/null http://localhost:5000/ && break
  sleep 1
done
curl -s -o /dev/null -w 'mlflow=%{http_code}\n' http://localhost:5000/
```
The new mlflow server process inherits the corrected environment. The poll loop waits up to 30 s for the server to bind port 5000; the final `curl` then prints `mlflow=200` once it is up. A `mlflow=000` reading means the server has not bound the port yet — wait a few more seconds and retry the `curl`.

##### 6. Re-run the smoke-test.
```
python3 /root/code/log_test_run.py
```
Output: `test-remote run logged successfully`.

##### 7. Verify in the MLflow UI.
Refresh the **MLflow UI** tab. Open **Experiments → test-remote** — one or more runs are listed. Click the latest `remote-smoke-test` run; the **Artifacts** panel now shows a `model/` directory with the logged files.

##### 8. Verify in the SeaweedFS Filer.
Open the **SeaweedFS Filer** tab and navigate to `/buckets/mlflow-artifacts/`. Drill into the experiment / run / `model/` path — `MLmodel`, `model.pkl`, `conda.yaml`, etc. are present.

Or list from the terminal:
```
AWS_ACCESS_KEY_ID=weedadmin AWS_SECRET_ACCESS_KEY=weedadmin123 \
  aws --endpoint-url=http://localhost:8333 s3 ls s3://mlflow-artifacts/ --recursive | head
```

##### 9. Spot-check the PostgreSQL schema.
```
docker exec mlflow-db psql -U mlflow -d mlflow -c "\\dt" | head -20
```
PostgreSQL lists the MLflow schema tables — the run's metadata lives there, not in any local SQLite file.

#### References
- MLflow tracking server — backend store vs artifact store, proxied artifacts: https://mlflow.org/docs/latest/tracking/server.html
- MLflow artifact stores — S3-compatible storage and `MLFLOW_S3_ENDPOINT_URL`: https://mlflow.org/docs/latest/tracking/artifact-stores.html
- MLflow backend stores (the PostgreSQL `--backend-store-uri`): https://mlflow.org/docs/latest/tracking/backend-stores.html
