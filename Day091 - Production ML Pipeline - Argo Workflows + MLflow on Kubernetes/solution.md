# Solution

A production pipeline spans several Argo resources — a `WorkflowTemplate` that trains and registers a model, and a `CronWorkflow` that re-runs it on a cadence — where a single stale reference in any one of them breaks the whole pipe. This capstone reads a broken release as symptoms on the running system, fixes three cross-boundary bugs entirely through the Argo UI's YAML editors, and confirms a new `fraud-detector` version reaches MLflow.

> As an MLOps engineer, you read failures across resource boundaries and fix wiring in place on a running system so the full train-register-retrain pipe passes again — you are not tuning the model; the training steps are synthetic stand-ins.

#### Follow the steps below

##### 1. Take stock.
From a VS Code terminal:
```
curl -s -o /dev/null -w 'argo=%{http_code}\n'   http://localhost:5000/
curl -s -o /dev/null -w 'mlflow=%{http_code}\n' http://localhost:5001/
kubectl -n mlflow get pods
kubectl -n argo get workflowtemplate,cronworkflow
```
Both UIs answer `200`, the MLflow pod is `Running`, and the two Argo resources exist.

##### 2. Submit the template — it is rejected on the parameter typo.
Click the **Argo UI** button. From the Workflows page click **+ Submit New Workflow → From Template → fraud-training-pipeline → + Submit**. The submit is rejected immediately with a validation error:
```
templates.main.steps failed to resolve {{steps.train.outputs.parameters.runid}}
```
Argo refuses to create the workflow: the `register` step references an output parameter `runid` that no step produces — `train` emits `run_id` (with an underscore). This has to be fixed before any run can start.

##### 3. Fix the parameter reference (`run_id`) — Argo UI Templates editor.
Left navigation → **Workflow Templates** → `fraud-training-pipeline` → **Edit** (pencil icon, top-right). Locate:
```yaml
  - name: register
    arguments:
      parameters:
        - name: run_id
          value: "{{steps.train.outputs.parameters.runid}}"
```
Change `runid` to `run_id`:
```yaml
          value: "{{steps.train.outputs.parameters.run_id}}"
```
Click **Save**.

##### 4. Submit again — the run starts and `train` fails on the MLflow DNS.
Back to Workflows → **+ Submit New Workflow → From Template → fraud-training-pipeline → + Submit**. The workflow is created this time and runs. The `train` node fails; its log ends with something like:
```
HTTPConnectionPool(host='mlflow.default.svc.cluster.local', port=5000):
  Max retries exceeded ... Name or service not known
```
MLflow lives in the `mlflow` namespace, not `default`.

##### 5. Fix the MLflow namespace on both steps — same Templates editor.
Edit `fraud-training-pipeline` again. Locate both occurrences of:
```yaml
    env:
      - name: MLFLOW_TRACKING_URI
        value: "http://mlflow.default.svc.cluster.local:5000"
```
(one on the `train` script template, one on the `register` script template).

Change both to:
```yaml
        value: "http://mlflow.mlflow.svc.cluster.local:5000"
```
**Save**. Re-submit. Both `train` and `register` go green. Workflow phase: `Succeeded`.

Check the MLflow UI (port `5001`) → **Models** → `fraud-detector` is listed with version `1`.

##### 6. Fix Bug 3 (CronWorkflow template name) — Argo UI CronWorkflows editor.
Left navigation → **Cron Workflows** → `fraud-retraining` → **Edit** (pencil icon).

Locate:
```yaml
spec:
  workflowSpec:
    workflowTemplateRef:
      name: training-pipeline
```
Change `training-pipeline` to `fraud-training-pipeline`:
```yaml
      name: fraud-training-pipeline
```
**Save**.

On its next tick (the schedule is every minute) the CronWorkflow fires a child Workflow, which appears on the Workflows list and under the cron's detail page. Give it a few minutes to finish — each step installs its Python deps on start, so a run is not instant — after which it reaches `Succeeded` and adds another `fraud-detector` version to the MLflow Models page. You do not need to submit anything by hand for the cron; it produces the run on schedule (a manual **+ Submit** also works if you would rather not wait for the tick).

##### 7. Verify via REST.
```
# Templates + cron now reference the right names.
curl -s http://localhost:5000/api/v1/workflow-templates/argo/fraud-training-pipeline \
  | python3 -c "
import json, sys
body = json.load(sys.stdin)
for t in body['spec']['templates']:
    if t.get('script'):
        env = next((e['value'] for e in (t['script'].get('env') or [])
                    if e['name']=='MLFLOW_TRACKING_URI'), '-')
        print(f\"{t['name']:10s}  MLFLOW_TRACKING_URI={env}\")
"

curl -s http://localhost:5000/api/v1/cron-workflows/argo/fraud-retraining \
  | python3 -c "
import json, sys
body = json.load(sys.stdin)
ref = body['spec']['workflowSpec']['workflowTemplateRef']['name']
print(f'cron workflowTemplateRef.name={ref}')
"

# MLflow has a version:
curl -s 'http://localhost:5001/api/2.0/mlflow/registered-models/get?name=fraud-detector' \
  | python3 -c "
import json, sys
versions = json.load(sys.stdin)['registered_model']['latest_versions']
print('versions:', [v['version'] for v in versions])
"
```
The output shows both `MLFLOW_TRACKING_URI` values targeting `mlflow.mlflow...`, the cron's `workflowTemplateRef.name` equal to `fraud-training-pipeline`, and at least one registered version of `fraud-detector`.

#### References

- Argo Workflows — WorkflowTemplates (`workflowTemplateRef`, submitting from templates): https://argo-workflows.readthedocs.io/en/latest/workflow-templates/
- Argo Workflows — CronWorkflows (`schedules`, `workflowSpec`, `workflowTemplateRef`): https://argo-workflows.readthedocs.io/en/latest/cron-workflows/
- MLflow — Model Registry (registered models and versions): https://mlflow.org/docs/latest/ml/model-registry/
