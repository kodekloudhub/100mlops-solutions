# Solution

Kubeflow Pipelines runs ML workflows as DAGs on Kubernetes: each `@dsl.component` becomes one container-per-step, and the `@dsl.pipeline` function wires the ordering between them. This task hands you a pipeline that only wires the first step, and you add the `train` component with `.after(prep_data)`, compile it to IR YAML, and run it to Succeeded through the KFP UI.

> As an MLOps engineer, you wire and orchestrate the pipeline DAG so steps run in the right order — you are not training a real model; the `prep_data` and `train` components are stand-ins for the workflow.

#### Follow the steps below

##### 1. Confirm KFP is up.
From a VS Code terminal:
```
kubectl get pods -n kubeflow
curl -s -o /dev/null -w 'kfp=%{http_code}\n' http://localhost:5000/
```
All `kubeflow` namespace pods are Running and the UI answers `200`. There is no `pipeline.yaml` yet — you compile it after completing the source.

##### 2. Complete the pipeline DAG.
Open `/root/code/kfp/pipeline.py`. The two `@dsl.component` functions are written, but the pipeline function wires only the first step:
```python
def fraud_training_pipeline():
    prep = prep_data()
    # TODO: wire train after prep
```
Add the `train` component and order it after `prep_data`:
```python
def fraud_training_pipeline():
    prep = prep_data()
    train().after(prep)
```
`.after(prep)` declares the dependency edge — without it KFP would schedule both components in parallel. Save.

##### 3. Compile the source to an IR YAML.
The `kfp` SDK (`kfp==2.7.0`) is installed. Run the file — its `__main__` block calls `compiler.Compiler().compile(...)`:
```
cd /root/code/kfp
python3 pipeline.py
ls -la pipeline.yaml
```
`pipeline.yaml` is written next to the source. This is the IR the KFP UI executes.

##### 4. Download the compiled pipeline to the local machine.
The KFP UI's file picker reads from the browser's OS, not the lab container — so `pipeline.yaml` has to live on the local machine before it can be uploaded.

In the VS Code Explorer (left panel), open `code/kfp/`, right-click `pipeline.yaml`, and choose **Download…**. Save it anywhere convenient (e.g. `~/Downloads/pipeline.yaml`).

##### 5. Upload the pipeline through the KFP UI.
Click the **KFP UI** button. From **Pipelines** click **+ Upload Pipeline** (top-right).
- **Pipeline name:** `fraud-training`.
- **File:** click **Choose file** and pick the `pipeline.yaml` just downloaded.
- Click **Create**.

The Pipelines list now shows `fraud-training` with one version.

##### 6. Create a run.
Click `fraud-training` → **+ Create Run**.
- **Experiment:** `Default`.
- Leave parameters at defaults.
- Click **Start**.

The run opens. Its DAG shows `prep-data` → `train` (the ordering you wired with `.after`), and both nodes transition through `Pending` → `Running` → a green tick. Final state: **Succeeded**.

##### 7. Verify via REST.
```
curl -s 'http://localhost:5000/apis/v2beta1/pipelines?page_size=100' \
  | python3 -c "
import json, sys
for p in json.load(sys.stdin).get('pipelines', []):
    print(p.get('display_name'), '|', p.get('pipeline_id'))
"

curl -s 'http://localhost:5000/apis/v2beta1/runs?page_size=50' \
  | python3 -c "
import json, sys
for r in json.load(sys.stdin).get('runs', []):
    name = (r.get('pipeline_spec', {}).get('pipeline_info', {}) or {}).get('name', '-')
    print(f\"{name:20s}  state={r.get('state'):12s}  display={r.get('display_name')}\")
"
```
The first call lists `fraud-training`. The second shows a run with `state=SUCCEEDED` under that pipeline name.

#### References

- KFP SDK — DSL reference (`@dsl.component`, `@dsl.pipeline`, `PipelineTask.after`): https://kubeflow-pipelines.readthedocs.io/en/stable/source/dsl.html
- Kubeflow Pipelines — project docs and examples (`compiler.Compiler().compile`, IR YAML): https://github.com/kubeflow/pipelines
