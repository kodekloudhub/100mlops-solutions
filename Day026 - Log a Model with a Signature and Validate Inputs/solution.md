# Solution

An MLflow **model signature** records a model's input and output schema — column names and types — alongside the model artifact. When a model is logged with a signature, MLflow **enforces** it at predict time: inputs are validated and coerced against the schema, and a request missing a required column is rejected with a clear error instead of silently producing a wrong answer. An **input example** stored next to the model documents the expected payload and lets MLflow infer types it cannot otherwise see. Attaching a signature is the cheapest guardrail between a model and the malformed requests a live endpoint inevitably receives. This task infers a signature from the training data and logs the model with it.

> As an MLOps engineer, you attach and enforce an input schema on a logged model so malformed requests are rejected at the boundary — you are not improving the model; the training frame and estimator are synthetic.

#### Follow the steps below

##### 1. Confirm the MLflow server is running.
The lab startup launched the MLflow tracking server. Open the **MLflow UI** button at the top of the lab to view the dashboard.
```
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:5000/
```
A `200` confirms the server is reachable.

##### 2. Open the scaffold.
Open `/root/code/log_with_signature.py` in the VS Code editor. It builds a named two-column frame `X` (`amount`, `num_txn`), fits a `DummyClassifier`, and computes `preds = model.predict(X)`. Two `# TODO` blocks sit inside the `with mlflow.start_run():` context.

##### 3. TODO 1 — infer the signature.
`infer_signature` reads the schema straight off the example inputs and outputs — here the column names and dtypes of `X`, and the type of `preds`:
```python
    signature = infer_signature(X, preds)
```

##### 4. TODO 2 — log the model with the signature and an input example.
Pass both `signature` and `input_example` to `log_model`. The input example is persisted next to the model and also helps MLflow pin down column types:
```python
    mlflow.sklearn.log_model(
        model,
        name="model",
        signature=signature,
        input_example=X,
    )
```

##### 5. Run the script.
```
python3 /root/code/log_with_signature.py
```
It prints `model logged to the fraud-signature experiment` and exits, creating one run in the `fraud-signature` experiment.

##### 6. Inspect the signature in the MLflow UI (this is the *schema*).
Open the **MLflow UI** button → switch the toggle to **Model training** → **Experiments** → `fraud-signature` → click the run → the **Artifacts** tab. The logged model directory lists `MLmodel`, `model.skops` (the serialized estimator — MLflow's safe default format), `input_example.json`, `requirements.txt`, and more. Select **`MLmodel`** — its `signature:` block is the enforced input/output schema:
```yaml
signature:
  inputs: '[{"type": "double", "name": "amount", "required": true},
            {"type": "long", "name": "num_txn", "required": true}]'
  outputs: '[{"type": "tensor", "tensor-spec": {"dtype": "int64", "shape": [-1]}}]'
```
The UI shows you the **contract** (exactly the `amount` + `num_txn` inputs). It does **not**, however, show the *act* of enforcement — that only happens at predict time.

##### 7. Verify the schema is *enforced* (from a terminal).
Enforcement is runtime behaviour: it fires when something calls `.predict()` with a bad payload, so there is no UI screen for it. Load the model back and confirm it accepts a well-formed frame but rejects one missing a required column:
```
python3 -c "
import mlflow, pandas as pd
from mlflow import MlflowClient
mlflow.set_tracking_uri('http://localhost:5000')
c = MlflowClient()
exp = c.get_experiment_by_name('fraud-signature')
run = c.search_runs([exp.experiment_id], order_by=['attributes.start_time DESC'], max_results=1)[0]
m = mlflow.pyfunc.load_model('runs:/' + run.info.run_id + '/model')
print('signature:', m.metadata.signature)
print('good ->', m.predict(pd.DataFrame({'amount':[10.0],'num_txn':[2]})))
try:
    m.predict(pd.DataFrame({'amount':[10.0]}))
    print('bad  -> NOT enforced')
except Exception as e:
    print('bad  -> rejected:', type(e).__name__)
"
```
The good frame returns a prediction; the frame missing `num_txn` raises an `MlflowException` — the signature is doing its job. (In short: the **UI shows the schema**; this **terminal check proves the enforcement**.)

#### References
- MLflow Models — model signature and input example (schema enforcement): https://mlflow.org/docs/latest/model/signatures.html
- `mlflow.models.infer_signature`: https://mlflow.org/docs/latest/python_api/mlflow.models.html
- MLflow Models overview (the `MLmodel` file, pyfunc flavor): https://mlflow.org/docs/latest/models.html
