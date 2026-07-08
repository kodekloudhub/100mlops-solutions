# Solution

An MLflow **run** is one recorded execution of training/evaluation code. Inside a `with mlflow.start_run():` block you attach three kinds of data to that run: **parameters** — the inputs and hyperparameters (`mlflow.log_params`); **metrics** — numeric results (`mlflow.log_metric`); and **artifacts** — files and serialised models (`mlflow.sklearn.log_model`). MLflow stores parameters and metrics in the backend store and the model files in the artifact store, and the UI then shows all three under the run. This task completes the three logging calls in an otherwise-finished script.

> As an MLOps engineer, you make a training run fully traceable — params, metrics, and the model artifact all recorded on the run — you are not improving the model; the estimator is a deterministic toy and the metrics are synthetic.

#### Follow the steps below

##### 1. Confirm the MLflow server is already running.
The lab startup launched the MLflow tracking server in the background — no extra start step is required. Open the **MLflow UI** button at the top of the lab and verify that the dashboard loads with the `Default` experiment listed.

From a VS Code terminal:
```
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:5000/
```
A `200` confirms the server is reachable.

##### 2. Open `/root/code/log_experiment.py` in the VS Code editor.
The scaffolding around the MLflow calls is already in place: a `params` dict, a trivial sklearn estimator (`DummyClassifier` fitted on a small deterministic fixture — synthetic, not a real training run), and `accuracy` and `f1` scores computed from that model's predictions on the fixture (deterministic: `accuracy=0.75`, `f1_score≈0.857`). Scroll to the `with mlflow.start_run():` block — three `# TODO` comments mark the work.

##### 3. TODO 1 — log the hyperparameters.
`params` is a dictionary holding `n_estimators`, `max_depth`, and `random_state`. Use `mlflow.log_params()` to record every key and value as a run parameter in a single call:
```python
    mlflow.log_params(params)
```

##### 4. TODO 2 — log the metrics.
Use `mlflow.log_metric()` to record one metric at a time under a chosen key:
```python
    mlflow.log_metric("accuracy", accuracy)
    mlflow.log_metric("f1_score", f1)
```

##### 5. TODO 3 — log the trained model.
Use `mlflow.sklearn.log_model()` to serialise the estimator, write the `MLmodel` descriptor, and publish both under the run's artefact directory. In MLflow 3.x, pass the artefact directory name via `name=` — the earlier `artifact_path=` keyword is deprecated:
```python
    mlflow.sklearn.log_model(model, name="model")
```

##### 6. Save the file and run the script.
```
python3 /root/code/log_experiment.py
```
The computed accuracy and F1 print to stdout and the process exits. A new run is now recorded on the MLflow server.

##### 7. Verify in the MLflow UI.
Open the **MLflow UI** button → `Default` experiment → click the newest run. Three panels confirm the result:
- **Parameters** lists `n_estimators`, `max_depth`, and `random_state`.
- **Metrics** lists `accuracy` and `f1_score`.
- **Artifacts** contains the logged model entity with `MLmodel`, a pickled sklearn estimator, and a `requirements.txt`.

#### References
- `mlflow.log_params` / `mlflow.log_metric` (logging API): https://mlflow.org/docs/latest/python_api/mlflow.html
- `mlflow.sklearn.log_model` (model logging, `name=`): https://mlflow.org/docs/latest/python_api/mlflow.sklearn.html
- MLflow tracking — logging runs, params, metrics, artifacts: https://mlflow.org/docs/latest/tracking.html
