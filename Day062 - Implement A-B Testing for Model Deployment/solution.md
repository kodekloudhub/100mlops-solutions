# Solution

A/B testing serves two model versions side by side and splits live traffic between them so their real-world behaviour can be compared. This task authors the routing logic in a Flask server: a fixed 80/20 random split between two model pickles, tagging every response with the `model_version` that produced it so downstream systems can attribute each prediction.

> As an MLOps engineer, you split traffic and label every prediction by version so candidates can be compared on live data — you are not judging which model is statistically better here; the split-and-tag mechanism is the skill. The models and data are synthetic.

#### Follow the steps below

##### 1. Confirm the starting state.
From a VS Code terminal:
```
cd /root/code/serving
ls
cat ab_server.py
```
`model_v1.pkl`, `model_v2.pkl`, and `ab_server.py` are staged. In `ab_server.py` the model loads, `/health`, and the request-body parsing are wired; the routing logic in `/predict` is a `# TODO` that returns `501`.

##### 2. Author the routing logic.
Open `/root/code/serving/ab_server.py` in the VS Code editor. Replace the `# TODO` / `return jsonify({"error": ...}), 501` stub with the A/B routing — an 80/20 split, the chosen model's score, and a response that identifies the model:
```python
    if random.random() < 0.8:
        model = MODEL_V1
        version = "v1"
    else:
        model = MODEL_V2
        version = "v2"

    is_fraud = int(model.predict(features)[0])
    return jsonify({"is_fraud": is_fraud, "model_version": version}), 200
```
`random.random() < 0.8` sends ~80 % of traffic to `MODEL_V1`; the `model_version` field lets downstream monitoring attribute each prediction. Save the file.

##### 3. Start the server and verify.
```
python3 ab_server.py &
sleep 2
curl -s -X POST -H 'Content-Type: application/json' \
  -d '{"amount":500,"hour":12,"num_tx_past_day":2}' \
  http://localhost:8085/predict
```
The response now reads along the lines of `{"is_fraud": 0, "model_version": "v1"}` — the field is present.

##### 4. Sample the traffic split.
```
python3 -c "
import json, urllib.request
from collections import Counter
c = Counter()
for _ in range(200):
    req = urllib.request.Request(
        'http://localhost:8085/predict',
        data=json.dumps({'amount': 500, 'hour': 12, 'num_tx_past_day': 2}).encode(),
        headers={'Content-Type':'application/json'}, method='POST')
    with urllib.request.urlopen(req) as r:
        body = json.loads(r.read())
    c[body['model_version']] += 1
print(c)
"
```
Output reads along the lines of `Counter({'v1': 162, 'v2': 38})` — roughly 80 / 20 across 200 requests.

#### References

- Flask JSON responses with `jsonify` (the `model_version` field): https://flask.palletsprojects.com/en/stable/api/#flask.json.jsonify
- Python `random.random()` — the uniform draw the split is built on: https://docs.python.org/3/library/random.html#random.random
