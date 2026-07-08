# Solution

A multi-stage training **pipeline** chains independent steps — preprocess → featurize → train → evaluate — where each stage reads its **immediate predecessor's output**. The contract holds only if the wiring is right: every stage's *input* path must point at the *output* the previous stage wrote. When a stage reaches back to an earlier source, the chain silently breaks — work done upstream (here, dropping low-value rows in preprocess and engineering `amount_log` in featurize) never reaches the model, even though the pipeline still "runs" green. This task has **two** such broken links. Beyond the wiring, the **orchestrator** (`run_pipeline.py`) is what turns four independent scripts into one tracked experiment: an imperative Python runner that wraps the whole pipeline in a single MLflow run and records its parameters and metrics — the experiment-tracking layer a declarative pipeline definition alone does not provide. Two of its tracking steps are left for you to complete.

> As an MLOps engineer, you wire a multi-stage pipeline correctly and capture its end-to-end run in MLflow—you are not judging the model's predictive quality. The data is synthetic.

#### Follow the steps below

##### 1. Confirm the starting state.
Open the **MLflow UI** button at the top of the lab — the `training-pipeline` experiment is present and empty. From a VS Code terminal:
```
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:5000/
ls /root/code/fraud-detection/src/ /root/code/fraud-detection/run_pipeline.py /root/code/fraud-detection/configs/pipeline_config.yaml
```
A `200` confirms MLflow is reachable. The four stage scripts, the orchestrator, and the config are already staged.

##### 2. Run the draft pipeline and observe the gaps.
```
cd /root/code/fraud-detection && python3 run_pipeline.py
```
All four stages complete and one run is written to the `training-pipeline` experiment. Compare the row counts each stage reports:
```
wc -l data/raw/train.csv data/processed/train_clean.csv data/features/features.csv
```
The raw CSV is 200 rows (+ header). The processed CSV is shorter — preprocess dropped the low-amount rows and duplicates. But the features CSV is still 200 rows — the preprocess stage's work never reached the feature matrix. And the held-out set the train stage writes lacks the `amount_log` column, so train trained on the pre-featurize data. Two links are broken. In the MLflow UI, the run also carries no parameters and no metrics — the orchestrator's tracking is unfinished.

##### 3. Diagnose the two broken links.
Each stage's input path must point at its immediate predecessor's output. Read the `input_path` / `features_path` assignments against the config's `data:` block:
- `src/featurize.py` reads `config["data"]["raw_path"]` — it bypasses preprocess. Its output row count should match the *preprocessed* count, so it must read `processed_path`.
- `src/train.py` reads `config["data"]["processed_path"]` — it bypasses featurize, so the model never sees the engineered `amount_log`. It must read `features_path`.

The config's `data:` block already exposes both keys; the fixes are confined to the stage scripts (the config must stay intact).

##### 4. Fix the featurize input source.
In `/root/code/fraud-detection/src/featurize.py`, change the `input_path` line so featurize reads the preprocess stage's output:
```python
input_path = config["data"]["processed_path"]
```

##### 5. Fix the train input source.
In `/root/code/fraud-detection/src/train.py`, change the `features_path` line so train reads the featurize stage's output:
```python
features_path = config["data"]["features_path"]
```
Save both files.

##### 6. Complete the orchestrator — TODO 1 (log the parameters).
Open `run_pipeline.py`. Inside the `with mlflow.start_run(...)` block, at TODO 1, log the config-driven hyperparameters onto the run:
```python
        mlflow.log_param("model_type", config["model"]["type"])
        mlflow.log_param("n_estimators", config["model"]["n_estimators"])
        mlflow.log_param("max_depth", config["model"]["max_depth"])
```

##### 7. Complete the orchestrator — TODO 2 (log the metrics).
At TODO 2, after the stage loop, read the evaluation report the last stage wrote and log every metric onto the same run:
```python
        with open(config["output"]["report_path"]) as f:
            metrics = json.load(f)
        for key, value in metrics.items():
            mlflow.log_metric(key, value)
```
Save the file.

##### 8. Re-run the pipeline.
```
cd /root/code/fraud-detection && python3 run_pipeline.py
wc -l data/processed/train_clean.csv data/features/features.csv
head -1 data/features/test_set.csv
```
The processed and features CSVs now report the same row count (both < 200), and the persisted `test_set.csv` header includes `amount_log` — confirming train consumed the feature matrix. The stage chain holds end to end.

##### 9. Verify the artefacts.
```
cat /root/code/fraud-detection/reports/evaluation.json
ls -l /root/code/fraud-detection/models/model.pkl
```
The JSON carries `accuracy`, `f1`, and `roc_auc` as numeric values. The pickled model exists.

##### 10. Verify in the MLflow UI.
Open the **MLflow UI** button → `training-pipeline` experiment. One run named `full-pipeline` is listed with `params.model_type`, `params.n_estimators`, and `params.max_depth`, plus `metrics.accuracy`, `metrics.f1`, and `metrics.roc_auc`. The **Artifacts** tab lists `model.pkl`.

#### References
- MLflow logging (`start_run` / `log_param` / `log_metric` — the orchestrator's end-to-end run): https://mlflow.org/docs/latest/python_api/mlflow.html
- scikit-learn — pipelines and chaining estimators (the stage-chain idea): https://scikit-learn.org/stable/modules/compose.html
- PyYAML — `yaml.safe_load` (how each stage reads its config paths): https://pyyaml.org/wiki/PyYAMLDocumentation
