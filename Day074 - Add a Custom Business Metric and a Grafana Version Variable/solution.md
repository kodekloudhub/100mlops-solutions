# Solution

Beyond the metrics a framework emits for free, real platforms track **business** signals and slice dashboards by dimension. This task does both: instrument a custom Prometheus metric (`fraud_amount_usd_total`, a counter labelled by model `version`) into the serving app, then add a Grafana **template variable** (`$version`, sourced from `label_values(...)`) so one dashboard filters by model version. It is the counter → labelled series → `label_values` → `$variable` backbone of any multi-version ML dashboard.

> As an MLOps engineer, you instrument the metrics the business actually cares about and template dashboards so one board serves every model version — you are not modelling anything; the metric is synthetic.

#### Follow the steps below

##### 1. Confirm the starting state.
From a VS Code terminal:
```
docker ps --format '{{.Names}}\t{{.Status}}'
curl -s 'http://localhost:9090/api/v1/query?query=fraud_amount_usd_total' \
  | python3 -m json.tool
```
Three containers are running. The Prometheus query currently returns an empty `result` array — the custom counter does not exist yet.

##### 2. Add the custom counter to the emitter.
Open `/root/code/monitoring/app/metric_emitter.py` in the VS Code editor. After the existing `REQUEST_TOTAL` / `PREDICTION_ACCURACY` / `DATA_DRIFT_SCORE` / `INFERENCE_LATENCY` declarations, add:
```python
FRAUD_AMOUNT = Counter(
    "fraud_amount_usd_total",
    "Cumulative fraudulent-transaction amount (USD), by model version.",
    labelnames=["version"],
    registry=REGISTRY,
)
```
Inside the `_nudge_metrics` loop (the `for version in ("v1", "v1", "v1", "v2"):` block), add one line alongside the existing counter/histogram updates:
```python
FRAUD_AMOUNT.labels(version=version).inc(random.uniform(50, 500))
```
Save.

##### 3. Reload the emitter.
The file is bind-mounted into the container, so a restart is enough — no rebuild:
```
cd /root/code/monitoring
docker compose restart metric-emitter
```
Confirm Prometheus picks up the new series (may take up to 5 s for the first scrape):
```
curl -s 'http://localhost:9090/api/v1/query?query=fraud_amount_usd_total' \
  | python3 -m json.tool
```
The `result` array now carries one entry per `version` label (`v1`, `v2`).

##### 4. Log in to Grafana.
Click the **Grafana** button at the top of the lab. Enter:
- **Username:** `admin`
- **Password:** `grafana2026`

##### 5. Create the dashboard and the `version` variable.
From the left navigation:
- Click **Dashboards -> New -> New dashboard**. The new dashboard opens in edit mode.
- In the dashboard toolbar, click the **Dashboard options** icon (top right) → **Settings** → **Variables** tab → **Add variable**.
- In the **Select variable type** dropdown, pick **Query**.
- **Name:** `version`.
- Under **Query options**:
  - **Data source:** `Prometheus`.
  - **Query type:** `Classic query`.
  - **Classic query** field:
    ```
    label_values(flask_http_request_total, version)
    ```
- Leave **Regex** empty (the `.*-(?<text>.*)-(?<value>.*)-.*/` string is placeholder text, not a default value).
- Scroll down to **Preview of values** — it renders inline and should list `v1` and `v2`.
- Click **Back to list** (top of the Variables page) — the new `version` variable appears in the table.

A new dropdown labelled `version` now sits in the dashboard header.

##### 6. Add the panel that uses `$version`.
- Click **Add visualization** on the canvas.
- Data source: **Prometheus**.
- **Code** tab:
  ```
  rate(fraud_amount_usd_total{version="$version"}[1m])
  ```
- **Run queries**. The line plots the per-second fraud-amount rate for whichever version is selected in the dropdown.
- Right sidebar -> **Title:** `Fraud amount rate (USD/s)`.

##### 7. Save the dashboard.
- Click **Save dashboard** (disk icon).
- **Dashboard title:** `Fraud amount by version`.
- Click **Save**.

##### 8. Verify.
Flip the `version` dropdown between `v1` and `v2`. The panel re-queries with the new filter and the line shifts. From a VS Code terminal:
```
DASH_UID=$(curl -s -u admin:grafana2026 'http://localhost:3000/api/search?type=dash-db' \
  | python3 -c "import json, sys; print(json.load(sys.stdin)[0]['uid'])")
curl -s -u admin:grafana2026 http://localhost:3000/api/dashboards/uid/$DASH_UID \
  | python3 -c "
import json, sys
d = json.load(sys.stdin)['dashboard']
for v in d.get('templating', {}).get('list', []):
    q = v.get('query')
    if isinstance(q, dict):
        q = q.get('query')
    print(f\"var: {v.get('name'):10s} | query: {q}\")
for p in d.get('panels', []):
    expr = (p['targets'][0].get('expr') if p.get('targets') else '-')
    print(f\"panel: {p.get('title'):30s} | {expr}\")
"
```
The output confirms the `version` variable with its `label_values(...)` query and the panel expression that references `fraud_amount_usd_total` and `$version`.

#### References

- Prometheus Python client — `Counter` with label names: https://prometheus.github.io/client_python/instrumenting/counter/
- Grafana — dashboard (template) variables: https://grafana.com/docs/grafana/latest/dashboards/variables/
- Grafana — add a query variable with `label_values(...)`: https://grafana.com/docs/grafana/latest/dashboards/variables/add-template-variables/
