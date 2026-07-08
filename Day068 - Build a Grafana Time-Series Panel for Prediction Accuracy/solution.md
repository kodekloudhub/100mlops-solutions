# Solution

With Grafana wired to Prometheus, the next step is visualizing a live model signal. This task builds a Grafana **time-series** panel that plots `prediction_accuracy` — a rolling model-quality metric scraped from the serving stack — so decay shows up as a falling line rather than a surprise in production. It's the base panel skill (query → visualize → save) the rest of the section composes on.

> As an MLOps engineer, you put the model's live quality metric on a chart so degradation is visible at a glance — you are not evaluating the model offline; the accuracy series is synthetic.

#### Follow the steps below

##### 1. Confirm the starting state.
From a VS Code terminal:
```
docker ps --format '{{.Names}}\t{{.Status}}'
curl -s -u admin:grafana2026 http://localhost:3000/api/datasources | python3 -m json.tool
```
All three containers (`metric-emitter`, `mon-prometheus`, `mon-grafana`) are running. The datasources response lists one Prometheus entry — pre-provisioned at startup.

##### 2. Log in to Grafana.
Click the **Grafana** button at the top of the lab. Enter:
- **Username:** `admin`
- **Password:** `grafana2026`

##### 3. Open a new dashboard.
From the left navigation:
- Click **Dashboards** (the four-squares icon).
- Click **New → New dashboard** (top right).

The empty canvas opens with a prompt to **Add visualization**.

##### 4. Add a visualization panel.
Click **Add visualization**. Grafana asks which data source to use — select **Prometheus** (the only one listed; pre-provisioned).

##### 5. Query `prediction_accuracy`.
The query editor opens. In the **Code** tab (top-right of the editor), enter:
```
prediction_accuracy
```
Click **Run queries** (or press **Shift + Enter**). The main canvas renders a line chart centred around 0.85, drifting up and down as the metric-emitter's background thread nudges the gauge. The visualization is already set to **Time series** (the default for new panels).

##### 6. Title the panel.
On the right sidebar, set:
- **Title:** `Prediction accuracy`

Leave every other option at its default.

##### 7. Save the dashboard.
Click the **Save dashboard** icon (disk) at the top-right. Grafana asks for a dashboard title:
- **Dashboard title:** `Model performance`

Click **Save**. The dashboard reloads with the saved title.

##### 8. Verify via Grafana's API.
From a VS Code terminal:
```
curl -s -u admin:grafana2026 'http://localhost:3000/api/search?type=dash-db' | python3 -m json.tool
DASH_UID=$(curl -s -u admin:grafana2026 'http://localhost:3000/api/search?type=dash-db' \
  | python3 -c "import json, sys; print(json.load(sys.stdin)[0]['uid'])")
curl -s -u admin:grafana2026 http://localhost:3000/api/dashboards/uid/$DASH_UID | python3 -m json.tool
```
The search returns the dashboard you saved. The detail JSON shows the panel with `"type": "timeseries"` and a Prometheus target whose `expr` is `prediction_accuracy`.

#### References

- Grafana — the Time series visualization: https://grafana.com/docs/grafana/latest/panels-visualizations/visualizations/time-series/
- Grafana — build a dashboard and add a panel: https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/
- Prometheus querying basics (PromQL): https://prometheus.io/docs/prometheus/latest/querying/basics/
