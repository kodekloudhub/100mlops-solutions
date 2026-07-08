# Solution

Some models are too slow to answer inside a single HTTP request. The fix is asynchronous inference: the endpoint accepts the request, returns a `task_id` immediately, a background worker scores it, and the result is stored for later retrieval. This task authors the Redis round-trip — the worker's result `set` (with a TTL) and the `/result/<task_id>` lookup.

> As an MLOps engineer, you decouple slow scoring from the request path with a queue and a result store so the API stays responsive under load — you are not changing the model. The data is synthetic.

#### Follow the steps below

##### 1. Confirm the starting state.
From a VS Code terminal:
```
docker ps --filter name=async-redis
cd /root/code/serving
cat async_app.py
```
The `async-redis` container is running on host port 6379, and the Flask app's Redis client is pre-wired to it. Two things are `# TODO`s: the worker's result store in `_run_prediction`, and the `/result/<task_id>` handler (which currently only echoes the `task_id`).

##### 2. Author the worker's result store.
Open `/root/code/serving/async_app.py` in the VS Code editor. In `_run_prediction`, persist the classification to Redis so `/result` can retrieve it:
```python
def _run_prediction(task_id: str, features) -> None:
    time.sleep(0.3)
    is_fraud = int(MODEL.predict(np.array([features]))[0])
    REDIS.set(RESULT_KEY.format(task_id=task_id), is_fraud, ex=RESULT_TTL_SECONDS)
```
The key is shaped `result:<task_id>` with a 600-second TTL — so a client that polls after the worker finishes finds the result, and stale results expire.

##### 3. Author the `/result` lookup.
Read the stored value back from Redis and return it:
```python
@app.route("/result/<task_id>")
def result(task_id):
    stored = REDIS.get(RESULT_KEY.format(task_id=task_id))
    if stored is None:
        return jsonify({"task_id": task_id, "status": "pending"}), 202
    return jsonify({"task_id": task_id, "is_fraud": int(stored)}), 200
```
`REDIS.get` returns `None` until the worker has stored the result — the handler surfaces a `pending` status while the prediction is still in flight, then the `is_fraud` label once it lands. Save the file.

##### 4. Start the server and verify end-to-end.
```
cd /root/code/serving
python3 async_app.py &
sleep 2

task=$(curl -s -X POST -H 'Content-Type: application/json' \
  -d '{"amount":3200,"hour":23,"num_tx_past_day":5}' \
  http://localhost:8085/predict-async | python3 -c 'import json,sys;print(json.load(sys.stdin)["task_id"])')
echo "task_id: $task"

sleep 1
curl -s http://localhost:8085/result/$task
```
`POST /predict-async` returns immediately with a `task_id`; after the worker finishes, `GET /result/<task_id>` reads the stored value and returns `{"task_id":"...","is_fraud":1}`. The HTTP entrypoint stayed fast while the model ran on the background worker.

#### References

- redis-py — the `set` / `get` client calls used for the result store: https://redis.io/docs/latest/develop/clients/redis-py/
- Redis `SET` with expiry (`EX`) — the per-task TTL: https://redis.readthedocs.io/en/stable/commands.html
