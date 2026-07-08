# Solution

MLflow **autologging** replaces hand-written `log_param` / `log_metric` / `log_model` boilerplate with a single call that instruments a framework. After `mlflow.sklearn.autolog()`, MLflow hooks every subsequent `estimator.fit(...)` and automatically records **all** of the estimator's constructor parameters (not just the ones you passed), the training metrics it computes, and the fitted model as an artifact — into whatever experiment is active. Two ordering rules matter: autolog must be enabled, and the target experiment selected with `mlflow.set_experiment(...)`, **before** `fit()` runs. This task turns autolog on and routes the run into a named experiment.

> As an MLOps engineer, you replace hand-written logging with autolog so every run captures its full parameter set, metrics, and model artifact automatically — you are not improving the model; the estimator fits a deterministic toy fixture.

#### Follow the steps below

##### 1. Confirm the MLflow server is running.
The lab startup launched the MLflow tracking server — no extra start step is required. Open the **MLflow UI** button at the top of the lab; only the `Default` experiment is listed on first load.

```
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:5000/
```
A `200` confirms the server is reachable.

##### 2. Open the scaffold.
Open `/root/code/autolog_experiment.py` in the VS Code editor. The script configures the tracking URI, defines a deterministic four-row synthetic dataset, instantiates a `LogisticRegression`, and calls `model.fit(X, y)`. Two `# TODO` blocks sit above the dataset definition.

##### 3. TODO 1 — enable autologging for sklearn.
A single line turns on auto-capture. MLflow has several autolog flavours (`sklearn`, `xgboost`, `pytorch`, and so on). The sklearn flavour instruments every future `model.fit()` call so that the estimator's constructor parameters, its training metrics, and the fitted model are recorded automatically:
```python
mlflow.sklearn.autolog()
```

##### 4. TODO 2 — set the active experiment.
Route the next run into an experiment named `autolog-demo`. If the experiment does not already exist, MLflow creates it on first call:
```python
mlflow.set_experiment("autolog-demo")
```

> Order matters. Call both `autolog()` and `set_experiment(...)` **before** `model.fit()`. Autolog hooks the sklearn estimator globally at call time; `set_experiment` scopes the active run. Once `fit()` has completed, no further configuration is honoured for that run.

##### 5. Run the script.
```
python3 /root/code/autolog_experiment.py
```
The script prints `Autolog run complete — check the MLflow UI` and exits.

##### 6. Verify in the MLflow UI.
Open the **MLflow UI** button → `autolog-demo` experiment → click the newest run. Three panels confirm the result:
- **Parameters** lists a dozen or more entries (`C`, `max_iter`, `solver`, `tol`, `penalty`, `fit_intercept`, `random_state`, ...) — every sklearn default captured by autolog, not merely the three arguments the scaffold passes to the constructor.
- **Metrics** contains `training_accuracy_score`, `training_f1_score`, `training_precision_score`, and `training_recall_score` — metrics computed by autolog on the training set.
- **Artifacts** contains the logged model entity with an `MLmodel` descriptor and a pickled sklearn estimator.

#### References
- MLflow automatic logging (overview, supported flavours): https://mlflow.org/docs/latest/tracking/autolog.html
- `mlflow.sklearn.autolog` (parameters and behaviour): https://mlflow.org/docs/latest/python_api/mlflow.sklearn.html
- `mlflow.set_experiment` (selecting the active experiment): https://mlflow.org/docs/latest/python_api/mlflow.html
