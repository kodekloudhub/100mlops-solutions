# Solution

DVC can track small JSON/YAML files as *metrics* — model scores such as accuracy or F1 — separately from regular outputs. Declaring a file under a stage's `metrics:` block (instead of `outs:`) lets `dvc metrics show` display the values and `dvc metrics diff` compare them across commits. Adding `cache: false` keeps the metrics file in Git rather than the DVC cache, so the small JSON is versioned alongside the code and its history is readable in diffs. A given file is either a regular output **or** a metric — not both.

> As an MLOps engineer, you wire the pipeline so DVC surfaces metrics files for tracking and diffing — you are not judging whether the scores are good; the metric values are synthetic.

#### Follow the steps below

##### 1. Observe the failure.
Move into the project, run the pipeline, and try to display the metrics.
```
cd /root/code/fraud-detection
dvc repro
dvc metrics show
```
The pipeline runs to completion and `metrics.json` appears on disk, but `dvc metrics show` reports no metric files. The cause is in `dvc.yaml` — `metrics.json` is currently listed under the `train` stage's regular `outs:` block, so DVC treats it as a generic file output rather than a metric.

##### 2. Inspect the pipeline definition.
```
cat dvc.yaml
```
The `train` stage produces both `models/model.pkl` and `metrics.json`, but they are both under `outs:`. The metrics file needs its own `metrics:` block so DVC can surface it through `dvc metrics show`.

##### 3. Move `metrics.json` to a `metrics:` block.
Redefine the `train` stage so `metrics.json` is declared under `metrics:` (with `cache: false`) instead of `outs:`. The `cache: false` flag keeps the metrics file in Git history rather than the DVC cache, which makes diffs across commits easy to read. The simplest way is to re-add the stage with `dvc stage add --force`, where `-M` declares a metric output that is *not* cached (equivalent to `cache: false`):
```
dvc stage add --force -n train \
  -d data/processed/train.csv -d src/models/train.py \
  -o models/model.pkl -M metrics.json \
  "python3 src/models/train.py"
```
This rewrites only the `train` stage in `dvc.yaml`, leaving `process_data` and `split_data` untouched. Confirm the result:
```
cat dvc.yaml
```
The `train` stage should now read:
```yaml
  train:
    cmd: python3 src/models/train.py
    deps:
      - data/processed/train.csv
      - src/models/train.py
    outs:
      - models/model.pkl
    metrics:
      - metrics.json:
          cache: false
```
You can also make this edit by hand in the editor — move `metrics.json` out of `outs:` and into a `metrics:` block as shown above.

##### 4. Re-run the pipeline so the registration takes effect.
```
dvc repro
```

##### 5. Verify.
Confirm DVC now sees the metrics and the values are sensible.
```
dvc metrics show
cat metrics.json
```
Once you start iterating on the model, `dvc metrics diff` compares these values across Git commits.

---

**References:**
- [DVC — `dvc stage add`](https://dvc.org/doc/command-reference/stage/add) (documents `--force` and `-M` / `--metrics-no-cache`)
- [DVC — `dvc metrics`](https://dvc.org/doc/command-reference/metrics)
- [DVC — Defining Pipelines (`metrics:` in dvc.yaml)](https://dvc.org/doc/user-guide/pipelines/defining-pipelines)
