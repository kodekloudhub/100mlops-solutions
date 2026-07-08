# Solution

An MLflow **pyfunc `PythonModel`** lets you ship a model *together with* the Python logic that surrounds it — preprocessing, post-processing, business rules — as one artifact with a uniform `.predict(context, model_input, params)` contract. Here the `ScaledPredictor` wrapper standardises each input column (subtract the mean, divide by the std) *before* delegating to the underlying estimator, so callers send raw features and the wrapper handles scaling internally. The lab has three pieces of work: author that preprocessing-and-delegate step inside `predict`, load the registry **champion** (resolved by alias, not a hard-coded version) as the inner model, and run a batch over the input file to produce a predictions CSV.

> As an MLOps engineer, you wrap a registry model in a pyfunc that owns its preprocessing and load the champion by alias so downstream batch jobs stay decoupled from version numbers — you are not evaluating model quality; the model and inputs are synthetic.

#### Follow the steps below

##### 1. Confirm the MLflow server is running.
The lab startup launched the MLflow tracking server and registered `fraud-detector` with a `champion` alias on version 1. Open the **MLflow UI** button at the top of the lab → **Models** → **fraud-detector** — the alias badge is visible next to version 1.

```
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:5000/
```
A `200` confirms the server is reachable.

##### 2. Open the scaffold.
Open `/root/code/predict_with_preprocessing.py` in the VS Code editor. The `ScaledPredictor` constructor and the `.predict()` signature are defined, but the `predict` body is left as TODO 1. The scaffold also loads the pre-staged inputs (`/root/code/data/inputs.csv`), computes mean and std, and instantiates the predictor. Three `# TODO` blocks remain.

##### 3. TODO 1 — author the preprocessing inside `predict`.
The wrapper stores the per-column `mean` and `std` on the instance. Inside `predict`, scale the input array with them and hand the scaled array to the inner model. `model_input` has already been turned into a float array `X`:
```python
    def predict(self, context, model_input, params=None):
        X = np.asarray(model_input, dtype=float)
        scaled = (X - self.mean) / self.std
        return self.model.predict(scaled)
```
This is the pyfunc contract in action: callers pass raw features, and the wrapper scales them the same way every time before the underlying model sees them.

##### 4. TODO 2 — load the champion model from the registry.
`mlflow.pyfunc.load_model()` resolves a `models:/<name>@<alias>` URI into an in-memory pyfunc whose `.predict()` delegates to the underlying sklearn estimator. The URI is already stored as the module-level constant `MODEL_URI`:
```python
inner_model = mlflow.pyfunc.load_model(MODEL_URI)
```

##### 5. TODO 3 — run the batch prediction and persist the result.
The `predictor` instance has been built above. Its `.predict(context, model_input, params=None)` signature matches the pyfunc contract; calling it directly runs scaling and delegation. The output is an array of predictions whose length matches the input row count:
```python
inputs["prediction"] = predictor.predict(None, inputs.values)
inputs.to_csv(OUTPUT_CSV, index=False)
```

##### 6. Run the script.
```
python3 /root/code/predict_with_preprocessing.py
```
No errors are printed. `/root/code/predictions.csv` is created with the original two input columns plus a new `prediction` column.

##### 7. Verify.
```
head -5 /root/code/predictions.csv
wc -l /root/code/predictions.csv
```
The first command shows the header (`feature_a,feature_b,prediction`) and a few data rows; the second returns `11` (one header plus ten data rows).

#### References
- MLflow `mlflow.pyfunc` — custom `PythonModel` and the `predict` contract: https://mlflow.org/docs/latest/python_api/mlflow.pyfunc.html
- MLflow Models — storage format, pyfunc flavor, loading: https://mlflow.org/docs/latest/models.html
- MLflow Model Registry — resolving versions by alias (`models:/name@alias`): https://mlflow.org/docs/latest/model-registry.html
