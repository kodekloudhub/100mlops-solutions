# Solution

Argo Workflows is a Kubernetes-native engine that runs each pipeline as a `Workflow` object — a graph of container steps the controller schedules, retries, and reports on. This task authors your first `Workflow` and submits it through the Argo UI's **+ Submit New Workflow** editor, watching a single stand-in training step run to `Succeeded`.

> As an MLOps engineer, you author a `Workflow` spec — `entrypoint`, `templates`, `container` — so a training step runs as an observable, retryable Kubernetes object; you are not tuning the model, the step is a synthetic `echo` stand-in.

#### Follow the steps below

##### 1. Confirm the starting state.
From a VS Code terminal:
```
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:5000/
kubectl -n argo get deploy
```
The Argo UI answers `200`. Both `workflow-controller` and `argo-server` deployments show `AVAILABLE 1/1`.

##### 2. Open the Argo UI.
Click the **Argo UI** button at the top of the lab. The **Workflows** page loads with an empty list.

##### 3. Author and submit your first Workflow.
- Click **+ Submit New Workflow** (top-right).
- The in-browser YAML editor opens with a default template that references an image the cluster's containerd no longer pulls. Replace the editor contents with a Workflow you author — a single container step on a current image running a stand-in training step:
  ```yaml
  apiVersion: argoproj.io/v1alpha1
  kind: Workflow
  metadata:
    generateName: train-hello-
    namespace: argo
  spec:
    entrypoint: train
    templates:
      - name: train
        container:
          image: alpine:3.20
          command: [echo]
          args: ["fraud-detector: training step complete"]
  ```
  The pieces to understand: `entrypoint` names which template runs first; `templates` is the list of steps; each `container` is a pod spec. `generateName` lets you submit repeatedly without name clashes.
- Click **+ Create**.

The UI navigates to the new workflow's detail page. One node on the DAG moves `Pending` → `Running` → `Succeeded` (green tick) in about 10 seconds.

##### 4. Inspect the pod log.
Click the green node. The right-hand drawer opens; click **Logs** → the `train` container prints:
```
fraud-detector: training step complete
```

##### 5. Verify via the API.
From a VS Code terminal:
```
curl -s http://localhost:5000/api/v1/workflows/argo \
  | python3 -c "
import json, sys
body = json.load(sys.stdin)
for w in body.get('items', []):
    meta = w.get('metadata', {})
    st = w.get('status', {}) or {}
    print(f\"{meta.get('name'):40s}  phase={st.get('phase')}\")
"
```
The output lists the new workflow with `phase=Succeeded`.

#### References

- Argo Workflows — core concepts (`Workflow`, `entrypoint`, `templates`, `container`): https://argo-workflows.readthedocs.io/en/latest/workflow-concepts/
- Argo Workflows — hello-world walk-through: https://argo-workflows.readthedocs.io/en/latest/walk-through/hello-world/
