# Solution

Flask and FastAPI make you wire the server yourself; BentoML is a framework built for model serving — it keeps a model store and turns a service class into a runnable, packageable API. This task registers the trained model in the store and authors a `@bentoml.service` class with a typed `@bentoml.api` predict method, then serves it with `bentoml serve`.

> As an MLOps engineer, you reach for a serving framework like BentoML so packaging, versioning, and the API surface are standardized instead of hand-rolled per project — you are not changing the model. The data is synthetic.

#### Follow the steps below

##### 1. Confirm the starting state.
From a VS Code terminal:
```
cd /root/code/serving
bentoml models list
cat service.py
```
`fraud_detector` is registered in the BentoML store. `service.py` uses the modern `@bentoml.service` class API: the model is declared with `bentoml.models.BentoModel(...)` and loaded in `__init__`; the `last_predictions` API is wired; the `predict` handler is a `# TODO` that returns an error.

##### 2. Author the predict handler.
Open `/root/code/serving/service.py` in the VS Code editor. Replace the `return {"error": ...}` stub with the scoring logic:
```python
    @bentoml.api
    def predict(
        self, amount: float, hour: int, num_tx_past_day: int
    ) -> Dict[str, Any]:
        features = np.array([[amount, hour, num_tx_past_day]])
        is_fraud = int(self.model.predict(features)[0])
        self._history.append({
            "amount": amount,
            "hour": hour,
            "num_tx_past_day": num_tx_past_day,
            "is_fraud": is_fraud,
        })
        return {"is_fraud": is_fraud}
```
The `@bentoml.api` method's typed parameters (`amount`/`hour`/`num_tx_past_day`) become the JSON request body and drive the Swagger schema. Save the file.

##### 3. Start the server.
```
cd /root/code/serving
bentoml serve service:FraudService --host 0.0.0.0 --port 3000 &
sleep 5
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:3000/
```
Once it prints `200`, open the **BentoML UI** button — the Swagger surface loads with `predict` and `last_predictions`.

##### 4. Verify scoring from the Swagger UI.
Expand `POST /predict` → **Try it out**, submit a high-value late-night payload, and **Execute**:
```json
{"amount": 3200, "hour": 23, "num_tx_past_day": 5}
```
The response is `{"is_fraud": 1}`. Submit a low-value daytime payload — `{"amount": 25.5, "hour": 10, "num_tx_past_day": 1}` — and it returns `{"is_fraud": 0}`. The two distinct payloads score differently.

##### 5. Cross-check the audit log.
```
curl -s -X POST -H 'Content-Type: application/json' -d '{}' \
  http://localhost:3000/last_predictions | python3 -m json.tool
```
Every prediction the handler recorded appears in the `predictions` array with its inputs and `is_fraud` label.

#### References

- BentoML services — the `@bentoml.service` class API and `@bentoml.api` methods: https://docs.bentoml.com/en/latest/build-with-bentoml/services.html
- Loading and managing models from the store: https://docs.bentoml.com/en/latest/build-with-bentoml/model-loading-and-management.html
- `bentoml serve` — running the HTTP server: https://docs.bentoml.com/en/latest/reference/cli.html
