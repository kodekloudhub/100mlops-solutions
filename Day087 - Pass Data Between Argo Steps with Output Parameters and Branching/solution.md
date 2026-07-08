# Solution

A `WorkflowTemplate` is a reusable pipeline definition that can be parameterised per run, letting one template serve dev, staging, and prod by only changing an argument. This task finishes a `train-and-maybe-register` template — publishing the evaluate score as an output parameter and gating the `register` step with a `when:` expression — then submits it twice to prove the gate both skips and fires.

> As an MLOps engineer, you pass a score between steps as an output parameter and gate model promotion with a configurable `when:` threshold so the same template serves every environment — you are not tuning the model; the evaluate score is a synthetic constant.

#### Follow the steps below

##### 1. Read the scaffolded template.
The startup staged an incomplete template on disk and did **not** apply it — the Argo UI's **Workflow Templates** list is empty. Open it in VS Code:
```
sed -n '1,60p' /root/code/argo/train-and-maybe-register.yaml
```
Two pieces are missing: the `evaluate` template does not publish its score (TODO 1), and the `register` step has no `when:` gate (TODO 2).

##### 2. Author TODO 1 — publish the evaluate score as an output parameter.
Under the `evaluate` template, add an `outputs.parameters` block that reads the file the script wrote:
```yaml
    - name: evaluate
      script:
        image: alpine:3.19
        command: [sh]
        source: |
          echo "0.75" > /tmp/score.txt
          echo "[evaluate] score=0.75"
      outputs:
        parameters:
          - name: score
            valueFrom:
              path: /tmp/score.txt
```
`valueFrom.path` turns a file the container wrote into a named output parameter other steps can reference.

##### 3. Author TODO 2 — gate the register step with `when:`.
On the `register` step inside the `main` template's `steps`, add the `when:` expression:
```yaml
        - - name: register
            template: register
            when: "{{=asFloat(steps.evaluate.outputs.parameters.score) > asFloat(workflow.parameters.min_score)}}"
```
`asFloat(...)` coerces the string parameters to numbers so the comparison is numeric, not lexical. Register now runs only when the evaluate score exceeds `min_score`. Save.

##### 4. Apply the template.
WorkflowTemplates are cluster resources — apply the finished file, then confirm it in the UI:
```
kubectl apply -n argo -f /root/code/argo/train-and-maybe-register.yaml
```
Click the **Argo UI** button → left nav **Workflow Templates** → `train-and-maybe-register` now appears. Click it.

##### 5. Submission 1 — `min_score` above the evaluate score.
On the template detail page, click **+ Submit** (top-right). The dialog opens with `min_score: 0.80` prefilled.
- Change `min_score` to `0.99` (above the `0.75` score).
- Click **+ Submit**.

The `main` steps render `train → evaluate → register`. `train` and `evaluate` go green; `register` stays dashed grey (**Skipped**) — the `when:` is false. Workflow phase: `Succeeded`.

##### 6. Submission 2 — `min_score` below the evaluate score.
Back on **Workflow Templates → `train-and-maybe-register`**, click **+ Submit** again.
- Change `min_score` to `0.5` (below the `0.75` score).
- Click **+ Submit**.

This run's `register` node is green — the `when:` is true. Workflow phase: `Succeeded`.

##### 7. Verify via the REST API.
From a VS Code terminal:
```
curl -s http://localhost:5000/api/v1/workflows/argo \
  | python3 -c "
import json, sys
body = json.load(sys.stdin)
for w in body.get('items', []):
    spec = w.get('spec', {})
    ref = spec.get('workflowTemplateRef', {})
    if ref.get('name') != 'train-and-maybe-register':
        continue
    args = spec.get('arguments', {}).get('parameters', [])
    ms = next((p['value'] for p in args if p['name'] == 'min_score'), '?')
    # Find the register node phase.
    reg_phase = '?'
    for n in (w.get('status') or {}).get('nodes', {}).values():
        nm = n.get('displayName') or n.get('name') or ''
        if nm.endswith('.register') or nm == 'register':
            reg_phase = n.get('phase'); break
    print(f\"min_score={ms:4s}  register.phase={reg_phase}\")
"
```
The output shows two lines — one with `min_score=0.99 register.phase=Skipped` (or `Omitted`) and one with `min_score=0.5 register.phase=Succeeded`.

#### References

- Argo Workflows — output parameters (`valueFrom.path`, `steps.X.outputs.parameters.Y`): https://argo-workflows.readthedocs.io/en/latest/walk-through/output-parameters/
- Argo Workflows — conditionals (`when:` with the `asFloat(...)` expr syntax): https://argo-workflows.readthedocs.io/en/latest/walk-through/conditionals/
- Argo Workflows — WorkflowTemplates and submitting from them: https://argo-workflows.readthedocs.io/en/latest/workflow-templates/
