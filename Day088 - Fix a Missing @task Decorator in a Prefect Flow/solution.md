# Solution

Prefect builds a flow-run graph from the task-run records its orchestrator emits, so a function must be registered as a `@task` to appear as a tracked node. This task fixes a `fraud-pipeline` flow whose `evaluate` step executes but never shows on the graph, redeploys the serve loop, and confirms a three-node `prep → train → evaluate` run.

> As an MLOps engineer, you register each pipeline step as a tracked task so the orchestrator has a run record for every step, not just the ones that happened to be decorated — you are not tuning the model; the steps are synthetic stand-ins.

#### Follow the steps below

##### 1. Confirm the Prefect server + serve loop are up.
From a VS Code terminal:
```
curl -s http://localhost:5000/api/health
curl -s http://localhost:5000/api/deployments/name/fraud-pipeline/fraud-pipeline \
  | python3 -c "import json, sys; d=json.load(sys.stdin); print('deployment:', d.get('name'), 'id:', d.get('id'))"
tail -3 /var/log/prefect-serve.log
```
Prefect's health endpoint answers `200`, the deployment is registered, and the serve loop log shows the worker polling for runs.

##### 2. Trigger the first Quick Run (broken).
Click the **Prefect UI** button. On the **Deployments** page, click `fraud-pipeline`. On the deployment's detail page, click the blue **Run ▶** button (top-right) → **Quick Run** from the dropdown. A toast appears with a link to the new Flow Run; alternatively go to **Runs** in the left nav and click the run that just appeared. On the Flow Run's detail page, the graph (under the **Task Runs** / Visualisation view) shows only two nodes (`prep` and `train`) — `evaluate` does not appear as a node even though the logs show its print statement firing.

##### 3. Restore the missing decorator on `evaluate`.
Open `/root/code/prefect/fraud_pipeline.py`. `prep` and `train` each carry a `@task(name=...)` decorator on the line directly above their `def` — that decorator is what makes Prefect register them as tracked nodes on the Flow Run graph. `evaluate` has no decorator. Add a matching one directly above `def evaluate(...)`:
```python
@task(name="evaluate")
def evaluate(model: str) -> float:
    print(f"[evaluate] scoring model {model}")
    return 0.75
```
Save. (Only the `@task(name="evaluate")` line is new — the function body is unchanged.)

##### 4. Redeploy.
```
cd /root/code/prefect
make redeploy
```
The helper kills the old serve process and restarts it, importing the fresh source. `/var/log/prefect-serve.log` confirms the new deployment is up.

##### 5. Trigger a second Quick Run.
Return to the Prefect UI → **Deployments** → click `fraud-pipeline` → **Run ▶** → **Quick Run**. Open the new Flow Run from the toast or via **Runs** in the left nav. The graph now renders three nodes in order: `prep → train → evaluate`. Click `evaluate` to see its state transitions and `Completed` result.

##### 6. Verify via the Prefect API.
From a VS Code terminal:
```
DID=$(curl -s http://localhost:5000/api/deployments/name/fraud-pipeline/fraud-pipeline \
  | python3 -c "import json, sys; print(json.load(sys.stdin)['id'])")

curl -s -X POST http://localhost:5000/api/flow_runs/filter \
  -H 'content-type: application/json' \
  -d "{\"flow_runs\":{\"deployment_id\":{\"any_\":[\"$DID\"]}},\"sort\":\"EXPECTED_START_TIME_DESC\",\"limit\":3}" \
  | python3 -c "
import json, sys
for r in json.load(sys.stdin):
    print(r['name'], r.get('state_type'))
"

FRID=$(curl -s -X POST http://localhost:5000/api/flow_runs/filter \
  -H 'content-type: application/json' \
  -d "{\"flow_runs\":{\"deployment_id\":{\"any_\":[\"$DID\"]},\"state\":{\"type\":{\"any_\":[\"COMPLETED\"]}}},\"sort\":\"EXPECTED_START_TIME_DESC\",\"limit\":1}" \
  | python3 -c "import json, sys; print(json.load(sys.stdin)[0]['id'])")

curl -s -X POST http://localhost:5000/api/task_runs/filter \
  -H 'content-type: application/json' \
  -d "{\"task_runs\":{\"flow_run_id\":{\"any_\":[\"$FRID\"]}}}" \
  | python3 -c "
import json, sys
for t in json.load(sys.stdin):
    print(t['name'], t.get('state_type'))
"
```
The first call lists flow runs; the second lists task runs for the latest completed one and shows three rows — `prep COMPLETED`, `train COMPLETED`, `evaluate COMPLETED`.

#### References

- Prefect — writing tasks (`@task`, why a step must be a task to be tracked): https://docs.prefect.io/v3/develop/write-tasks
- Prefect — writing flows (`@flow`, calling tasks): https://docs.prefect.io/v3/develop/write-flows
- Prefect — running flows via `.serve()` in a local process: https://docs.prefect.io/v3/deploy/run-flows-in-local-processes
