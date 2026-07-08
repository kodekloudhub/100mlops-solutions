# Solution

A **config-driven training** setup keeps every knob — which estimator, its hyperparameters, the dataset path, the target column, where to save the model — in a YAML file, so the training script stays fixed and a run is changed by editing config alone (no Python edits, easy to diff and reproduce). Here `train.py` reads `train_config.yaml`, resolves the estimator name through a small **registry** of allowed classes, trains, logs the run to MLflow, and serialises the model. When the config disagrees with reality — an estimator name the registry doesn't know, a target column the CSV doesn't contain, or an output path outside the project — the run either fails fast or drops its artifact in the wrong place. This task repairs those three mismatches in the config.

> As an MLOps engineer, you externalize a run's settings into config so it is reproducible and reconfigurable without touching code—you are not tuning the model or judging its accuracy. The dataset and model are synthetic.

#### Follow the steps below

##### 1. Confirm the starting state.
Open the **MLflow UI** button at the top of the lab — the dashboard loads with an empty `fraud-detection` experiment. From a VS Code terminal:
```
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:5000/
```
A `200` confirms the server is reachable.

##### 2. Attempt the training run and observe the failure.
```
python3 /root/code/fraud-detection/src/models/train.py
```
The script prints an `ERROR:` line — either the estimator name is unknown, or the target column is missing from the CSV. Depending on which check trips first, one of these two errors surfaces.

##### 3. Inspect the scaffold.
Open `/root/code/fraud-detection/src/models/train.py` in the VS Code editor. Confirm that every value is read from the YAML via `yaml.safe_load`, that the estimator is resolved through `ESTIMATOR_REGISTRY`, and that the registry only accepts the three full sklearn class names (`RandomForestClassifier`, `GradientBoostingClassifier`, `LogisticRegression`). This file is authoritative — do not modify it.

##### 4. Open the broken config.
Open `/root/code/fraud-detection/configs/train_config.yaml` in the VS Code editor. Three settings need correction.

**Fix 1 — estimator type.** The `model.type` field reads `RandomForest`. Change it to the full sklearn class name:
```yaml
model:
  type: RandomForestClassifier
```

**Fix 2 — target column.** The CSV's label column is `is_fraud`, not `target`. Change `data.target_column`:
```yaml
data:
  target_column: is_fraud
```

**Fix 3 — model output path.** The model must land inside the project tree at the absolute path `/root/code/fraud-detection/models/model.pkl`, not at the top level of `/root/code/`. Change `output.model_path`:
```yaml
output:
  model_path: /root/code/fraud-detection/models/model.pkl
```

Save the YAML.

##### 5. Re-run the trainer.
```
python3 /root/code/fraud-detection/src/models/train.py
```
The script prints `accuracy=...` and `f1_score=...` and finishes with `model saved to /root/code/fraud-detection/models/model.pkl`.

##### 6. Verify in the MLflow UI.
Open the **MLflow UI** button → `fraud-detection` experiment → click the newest run. Three panels confirm the result:
- **Parameters** — `model_type=RandomForestClassifier`, `n_estimators=100`, `max_depth=5`, `random_state=42`.
- **Metrics** — `accuracy` and `f1_score`.
- **Artifacts** — a `model/` directory containing `MLmodel`, a pickled sklearn estimator, and `requirements.txt`.

The file at `/root/code/fraud-detection/models/model.pkl` is the same serialised model:
```
ls -l /root/code/fraud-detection/models/model.pkl
```

#### References
- scikit-learn `RandomForestClassifier` (the exact estimator class names the registry expects): https://scikit-learn.org/stable/modules/generated/sklearn.ensemble.RandomForestClassifier.html
- PyYAML — `yaml.safe_load` and YAML structure: https://pyyaml.org/wiki/PyYAMLDocumentation
- MLflow logging API (`log_params` / `log_metric` / `log_model`): https://mlflow.org/docs/latest/python_api/mlflow.html
