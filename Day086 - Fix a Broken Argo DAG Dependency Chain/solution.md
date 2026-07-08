# Solution

An Argo `dag` template runs each task in its own pod and starts a task as soon as its `dependencies` are satisfied — so a task with no dependencies runs immediately. This task diagnoses a failed `data-prep → train → evaluate` DAG whose `evaluate` task has no dependency on `train`, so it starts in parallel and fails before the model exists, then resubmits a corrected spec so the three steps run as an ordered chain. The tasks share a workflow volume so the model file passes down the chain.

> As an MLOps engineer, you make the ordering contract between pipeline steps explicit so a downstream step never runs before its inputs exist — you are not tuning the model; the steps run synthetic stand-ins.

#### Follow the steps below

##### 1. Inspect the red run and read the failing step's log.
Click the **Argo UI** button at the top of the lab. The Workflows list shows `training-pipeline-<suffix>` in state **Failed**. Click it.

On the graph, `data-prep` and `train` are green; `evaluate` is red. Click the red `evaluate` node, then **LOGS** — the log (container `main`) shows exactly why it failed:
```
[evaluate] checking model
[evaluate] ERROR: model.pkl not found
```
`evaluate` ran before `train` produced `/workdir/model.pkl`.

##### 2. Confirm the cause in the spec.
The pipeline is staged on disk at `/root/code/pipelines/training-workflow.yaml` (it is the same spec that was submitted). Open it in the VS Code editor and look at the `main` template's `dag.tasks`:
- `data-prep` — no dependencies, runs first.
- `train` — `dependencies: [data-prep]`.
- `evaluate` — **no `dependencies:` field**. That is the bug: the DAG starts it immediately, in parallel with `data-prep` and `train`.

##### 3. Add the missing dependency.
Give the `evaluate` task the ordering contract it is missing — add a `dependencies: [train]` line so it waits for `train` to finish (the same mechanism `train` uses to run after `data-prep`):
```yaml
          - name: evaluate
            template: evaluate
            dependencies: [train]
```
Save.

##### 4. Submit the corrected workflow via the Argo UI.
From the Workflows list, click **+ Submit New Workflow** (top-right). The in-browser YAML editor opens with a default template — replace its contents with the fixed `/root/code/pipelines/training-workflow.yaml` (copy from VS Code, paste into the editor). Click **+ Create**.

##### 5. Watch the fixed DAG complete.
The UI opens the new workflow's detail page. The DAG now runs `data-prep → train → evaluate` in order; each node turns green in turn. Final phase: `Succeeded`.

##### 6. Verify via the API.
From a VS Code terminal:
```
curl -s http://localhost:5000/api/v1/workflows/argo \
  | python3 -c "
import json, sys
body = json.load(sys.stdin)
for w in sorted(body.get('items', []),
                key=lambda x: x['metadata']['creationTimestamp']):
    phase = (w.get('status') or {}).get('phase')
    name = w['metadata']['name']
    print(f'{name:40s}  phase={phase}')
"
```
The output lists two workflows: the original `Failed` and the new `Succeeded`.

#### References

- Argo Workflows — DAG template and task `dependencies`: https://argo-workflows.readthedocs.io/en/latest/walk-through/dag/
- Argo Workflows — sharing data between steps with `volumeClaimTemplates`: https://argo-workflows.readthedocs.io/en/latest/walk-through/volumes/
