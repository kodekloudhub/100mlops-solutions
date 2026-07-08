# Solution

`withParam` is Argo's fan-out primitive: one step definition becomes N parallel pods, one per input value, and a fan-in reducer step runs once they all finish. This task finishes a `train-parallel-variants` template — fanning `train-variant` out over a hyperparameter list and adding a `pick-best` reducer — then submits it twice to show both the failure mode and a clean sweep.

> As an MLOps engineer, you fan a training step out over an input list so branches run in parallel and a reducer picks the winner — you are not tuning the model; each branch runs a synthetic stand-in.

#### Follow the steps below

##### 1. Read the scaffolded template.
The startup staged an incomplete template on disk and did **not** apply it — the Argo UI's **Workflow Templates** list is empty. Open it in VS Code:
```
sed -n '1,40p' /root/code/argo/train-parallel-variants.yaml
```
The `train-variant` and `pick-best` templates are written, but the `main` template's `train` step has no `withParam` fan-out (TODO 1) and there is no `pick-best` fan-in step (TODO 2).

##### 2. Author TODO 1 — fan out with `withParam`.
On the `train` step in `main`, add the fan-out and pass each list item to the template:
```yaml
    - name: main
      steps:
        - - name: train
            template: train-variant
            arguments:
              parameters:
                - name: n_estimators
                  value: "{{item}}"
            withParam: "{{workflow.parameters.estimators_list}}"
```
`withParam` iterates the JSON list; `{{item}}` is the current element, handed to `train-variant` as its `n_estimators` input. One step definition becomes N parallel pods.

##### 3. Author TODO 2 — add the fan-in reducer step.
Add `pick-best` as a **second** step group so it runs after the whole fan-out completes:
```yaml
        - - name: pick-best
            template: pick-best
```
(The double-dash `- -` starts a new sequential step group; steps inside one group run in parallel, groups run in order.) Save.

**Order matters:** `train` (the fan-out) must be the **first** step group and `pick-best` (the reducer) the **second** — a reducer that runs before training has nothing to reduce. The finished `main` template reads exactly:
```yaml
    - name: main
      steps:
        - - name: train
            template: train-variant
            arguments:
              parameters:
                - name: n_estimators
                  value: "{{item}}"
            withParam: "{{workflow.parameters.estimators_list}}"
        - - name: pick-best
            template: pick-best
```

##### 4. Apply the template.
WorkflowTemplates are cluster resources — apply the finished file, then confirm it in the UI:
```
kubectl apply -n argo -f /root/code/argo/train-parallel-variants.yaml
```
Click the **Argo UI** button → left nav **Workflow Templates** → `train-parallel-variants` now appears. Click it.

##### 5. Submission 1 — provoke the failure.
On the template detail page click **+ Submit** (top-right). In the `estimators_list` parameter, replace the default with a list that includes one obviously-bad entry alongside two valid positive integers, then click **+ Submit**. On the workflow detail page:
- The DAG shows parallel `train-variant` pods.
- The pods for valid entries turn green.
- The pod for the bad entry turns red — click its Logs drawer to read the validation error.
- The `pick-best` reducer stays grey (Omitted); workflow phase: `Failed`.

##### 6. Submission 2 — clean list.
Back on the template detail page click **+ Submit** again. Set `estimators_list` to three valid positive integers and click **+ Submit**. All `train-variant` branches run in parallel and turn green; `pick-best` fires once they complete; workflow phase: `Succeeded`.

##### 7. Verify via the REST API.
From a VS Code terminal:
```
curl -s http://localhost:5000/api/v1/workflows/argo \
  | python3 -c "
import json, sys
body = json.load(sys.stdin)
for w in sorted(body.get('items', []),
                key=lambda x: x['metadata']['creationTimestamp'],
                reverse=True)[:4]:
    spec = w.get('spec') or {}
    if (spec.get('workflowTemplateRef') or {}).get('name') != 'train-parallel-variants':
        continue
    name = w['metadata']['name']
    phase = (w.get('status') or {}).get('phase')
    nodes = (w.get('status') or {}).get('nodes', {})
    trains = [n for n in nodes.values() if n.get('templateName') == 'train-variant']
    train_phases = [n['phase'] for n in trains if n.get('type') == 'Pod']
    pb = next((n for n in nodes.values() if n.get('templateName') == 'pick-best'), None)
    pb_phase = pb['phase'] if pb else '-'
    print(f\"{name}  wf={phase}  trains={train_phases}  pick_best={pb_phase}\")
"
```
The output shows two runs — the bad-list one with a mix of `Succeeded`/`Failed` train phases + `pick_best=Omitted` + workflow `Failed`, and the clean one with three `Succeeded` trains + `pick_best=Succeeded` + workflow `Succeeded`.

#### References

- Argo Workflows — loops (`withParam` over a list, `{{item}}`): https://argo-workflows.readthedocs.io/en/latest/walk-through/loops/
- Argo Workflows — steps (sequential step groups vs parallel steps): https://argo-workflows.readthedocs.io/en/latest/walk-through/steps/
- Argo Workflows — WorkflowTemplates and submitting from them: https://argo-workflows.readthedocs.io/en/latest/workflow-templates/
