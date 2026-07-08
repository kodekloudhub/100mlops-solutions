# Solution

Hyperparameters live in `params.yaml` so they can change without editing code. For DVC to *track* a parameter — and re-run a stage when that parameter changes — the stage must list the parameter under a `params:` section in `dvc.yaml`. Only then does DVC record the value in `dvc.lock` and treat a change to it as a reason to re-run the stage. A script can read `params.yaml` directly (as `train.py` does), but **without** the `params:` declaration DVC has no idea the stage depends on the value, so `dvc repro` will silently skip the stage when only the parameter changes.

> As an MLOps engineer, you wire a parameter into the pipeline so changing it triggers a re-run — you are not tuning the model for quality; the parameter and dataset are synthetic.

#### Follow the steps below

##### 1. Inspect the pipeline and the parameter.
```
cd /root/code/fraud-detection
cat params.yaml
sed -n '/train:/,$p' dvc.yaml
cat src/models/train.py
```
`train.py` reads `params["n_estimators"]`, and `params.yaml` declares `n_estimators: 100` — but the `train` stage in `dvc.yaml` has no `params:` section, so DVC is not tracking it.

##### 2. Wire the parameter into the `train` stage.
Add a `params:` section listing `n_estimators` to the `train` stage. Edit `dvc.yaml` directly so the `train` stage looks like this:
```yaml
  train:
    cmd: python3 src/models/train.py
    deps:
      - data/processed/train.csv
      - src/models/train.py
    params:
      - n_estimators
    outs:
      - models/model.pkl
```
> Equivalently: `dvc stage add --force -n train -d data/processed/train.csv -d src/models/train.py -p n_estimators -o models/model.pkl "python3 src/models/train.py"`.

##### 3. Run the pipeline.
```
dvc repro
```
DVC runs the pipeline and now records `n_estimators` for the `train` stage in `dvc.lock`.

##### 4. Demonstrate parameter-driven retraining.
Change the hyperparameter and re-run. Because the parameter is now tracked, DVC re-executes **only** the `train` stage.
```
sed -i 's/^n_estimators: 100/n_estimators: 200/' params.yaml
dvc repro
```
The output reports `Stage 'process_data' didn't change, skipping` and `Stage 'split_data' didn't change, skipping`, and re-runs `train`.

##### 5. Verify.
```
cat params.yaml
dvc params diff
ls -l models/model.pkl
```
`params.yaml` holds the new value, `dvc params diff` shows the change, and `models/model.pkl` has been regenerated. (Tip: had you skipped the `params:` wiring in step 2, this `dvc repro` would have reported "up to date" and **not** retrained — that is the whole point of tracking the parameter.)

---

**References:**
- [DVC — Parameters (`params`)](https://dvc.org/doc/command-reference/params)
- [DVC — Defining Pipelines (`params:` in dvc.yaml)](https://dvc.org/doc/user-guide/pipelines/defining-pipelines)
