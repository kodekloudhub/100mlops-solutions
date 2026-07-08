# Solution

Once a model is trained it has to answer requests. The simplest way is a small HTTP server: a Flask app that loads the model, exposes a `/predict` endpoint that reads a JSON request body, scores it, and returns the prediction as JSON. This task authors that handler and binds the server to the platform's forwarded port (8085, not Flask's default 5000).

> As an MLOps engineer, you wrap a trained model in a thin, stable HTTP contract so any client can call it the same way — you are not changing what the model predicts. The model and data are synthetic.

#### Follow the steps below

##### 1. Confirm the starting state.
From a VS Code terminal:
```
cd /root/code/serving
ls
cat app.py
```
`model.pkl`, `train.csv`, and `app.py` are staged. The Flask app binds to `port=5000`, `/health` and the model load are wired, and the `/predict` handler body is a `# TODO` that returns `501`.

##### 2. Run the draft server and observe the failures.
Start the draft server in the background:
```
python3 app.py &
sleep 2
```
The terminal shows `Running on http://0.0.0.0:5000`. Try the lab's exposed port:
```
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8085/health
```
`000` — nothing is listening on `8085`. Kill the draft server and clear the port:
```
pkill -f "python3 app.py"
```

##### 3. Fix the port.
Open `/root/code/serving/app.py` in the VS Code editor. Change the `app.run(...)` call:
```python
app.run(host="0.0.0.0", port=8085)
```
Save.

##### 4. Re-run and observe the unimplemented endpoint.
```
python3 app.py &
sleep 2
curl -s http://localhost:8085/health
```
Health returns `{"status":"ok"}`. Try a predict payload:
```
curl -s -X POST -H 'Content-Type: application/json' \
  -d '{"amount":3200,"hour":23,"num_tx_past_day":5}' \
  http://localhost:8085/predict
```
The response is `{"error":"predict not implemented"}` (HTTP 501) — the handler is still the scaffold stub.

##### 5. Author the `/predict` handler.
In `app.py`, replace the `# TODO` stub with a handler that reads the JSON body, builds the feature row, scores it, and returns the flag:
```python
@app.route("/predict", methods=["POST"])
def predict():
    payload = request.get_json() or {}
    amount = float(payload.get("amount", 0.0))
    hour = int(payload.get("hour", 0))
    num_tx_past_day = int(payload.get("num_tx_past_day", 0))
    features = np.array([[amount, hour, num_tx_past_day]])
    is_fraud = int(MODEL.predict(features)[0])
    return jsonify({"is_fraud": is_fraud}), 200
```
Save the file.

##### 6. Restart and verify.
```
pkill -f "python3 app.py"
python3 app.py &
sleep 2

curl -s -X POST -H 'Content-Type: application/json' \
  -d '{"amount":3200,"hour":23,"num_tx_past_day":5}' \
  http://localhost:8085/predict

curl -s -X POST -H 'Content-Type: application/json' \
  -d '{"amount":25.5,"hour":10,"num_tx_past_day":1}' \
  http://localhost:8085/predict
```
The two responses now differ — the endpoint is reading each payload and running the RandomForest against the submitted features.

#### References

- Flask request data — reading a JSON body with `request.get_json()`: https://flask.palletsprojects.com/en/stable/api/#flask.Request.get_json
- Flask `jsonify` — building JSON responses: https://flask.palletsprojects.com/en/stable/api/#flask.json.jsonify
- Flask quickstart — routing, methods, and `app.run(host, port)`: https://flask.palletsprojects.com/en/stable/quickstart/
