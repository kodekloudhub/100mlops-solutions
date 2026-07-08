# Solution

Individual panels answer one question each; an on-call engineer needs the whole model's health on one screen. This task composes a single **model-overview** dashboard from four panels spanning three visualization types (time-series, stat, bar gauge), pulling the serving and drift signals together so one glance tells you whether the model is healthy. The skill is dashboard composition and choosing the right visualization per signal.

> As an MLOps engineer, you build the one board the team opens during an incident — consolidating the signals that matter so triage is fast — you are not judging the model; the metrics are synthetic.

#### Follow the steps below

##### 1. Confirm the starting state.
From a VS Code terminal:
```
docker ps --format '{{.Names}}\t{{.Status}}'
curl -s -u admin:grafana2026 http://localhost:3000/api/datasources | python3 -m json.tool
curl -s 'http://localhost:9090/api/v1/query?query=data_drift_score' | python3 -m json.tool | head -30
cat /root/code/monitoring/drift/drift_scores.json
```
All three containers are running. The Prometheus datasource is pre-provisioned. The `data_drift_score` series carry one value per feature column — these are the PSI scores the **Evidently drift scorer** (`/root/code/monitoring/drift/drift_scorer.py`) recomputes every 15 seconds; `drift_scores.json` is the hand-off file the metric-emitter republishes from. Every ~minute the scorer also publishes a run to the Evidently workspace behind the **Evidently UI** button.

##### 2. Log in to Grafana.
Click the **Grafana** button at the top of the lab. Enter:
- **Username:** `admin`
- **Password:** `grafana2026`

##### 3. Open a new dashboard.
From the left navigation:
- Click **Dashboards** (four-squares icon).
- Click **New -> New dashboard** (top right).

##### 4. Panel 1 — Request rate (Time series).
- Click **Add visualization** on the empty canvas.
- Select **Prometheus** when Grafana asks which data source to use.
- In the **Code** tab, enter:
  ```
  sum(rate(flask_http_request_total[1m])) by (version)
  ```
- Click **Run queries**. Grafana plots a line per model version.
- Leave the **Visualization** as **Time series**.
- Right sidebar -> **Title:** `Request rate`.
- Click **Back to dashboard** (top left).

##### 5. Panel 2 — p95 inference latency (Time series).
- Click **Add -> Visualization**.
- Data source: **Prometheus**.
- **Code** tab:
  ```
  histogram_quantile(0.95, sum(rate(model_inference_duration_seconds_bucket[5m])) by (le))
  ```
- **Run queries**. The p95 line appears.
- Visualization: **Time series**.
- **Title:** `p95 inference latency (s)`.
- **Back to dashboard**.

##### 6. Panel 3 — Prediction accuracy (Stat).
- **Add -> Visualization**.
- Data source: **Prometheus**.
- **Code** tab:
  ```
  prediction_accuracy
  ```
- **Run queries**.
- On the right sidebar, change the **Visualization** from **Time series** to **Stat** — the panel collapses into a single-number readout.
- **Title:** `Prediction accuracy`.
- **Back to dashboard**.

##### 7. Panel 4 — Drift by column (Bar gauge).
- **Add -> Visualization**.
- Data source: **Prometheus**.
- **Code** tab:
  ```
  data_drift_score
  ```
- **Run queries** — three series appear, one per `column` label. Each value is the PSI Evidently computed for that feature against the reference window, so the bars move as the simulated production feed drifts.
- On the right sidebar, change the **Visualization** to **Bar gauge**. One horizontal bar per feature column renders.
- **Title:** `Drift by column`.
- **Back to dashboard**.

##### 8. Save the dashboard.
- Click the **Save dashboard** icon (disk, top-right).
- **Dashboard title:** `Model overview`.
- Click **Save**.

##### 9. Cross-check in the Evidently UI.
Click the **Evidently UI** button at the top of the lab (port `8000`):
- Click the `fraud-detector drift monitoring` project.
- The **Dashboard** tab (default) renders two pre-configured line charts — **Share of drifted columns** and **Per-column drift (PSI)** — one point per scoring run, accumulating every ~minute. This is Evidently's own monitoring dashboard over the exact same data your Grafana bar gauge shows live.
- The **Reports** tab lists the underlying runs; **View** any row → one card per feature renders, headed e.g. `Drift in column 'amount' — Data drift detected. Drift detection method: PSI. Drift score: 1.726`. Each card has two tabs: **Data Drift** (current mean ± std against the reference band) and **Data Distribution** (current-vs-reference histogram) — the raw evidence behind your Grafana bars.

##### 10. Verify via Grafana's API.
From a VS Code terminal:
```
DASH_UID=$(curl -s -u admin:grafana2026 'http://localhost:3000/api/search?type=dash-db' \
  | python3 -c "import json, sys; print(json.load(sys.stdin)[0]['uid'])")
curl -s -u admin:grafana2026 http://localhost:3000/api/dashboards/uid/$DASH_UID \
  | python3 -c "
import json, sys
d = json.load(sys.stdin)['dashboard']
for p in d['panels']:
    expr = (p['targets'][0].get('expr') if p.get('targets') else '-')
    print(f\"{p['type']:12s} | {p['title']:30s} | {expr}\")
"
```
The output lists four rows — one per panel — with distinct `type` values (at least three of `timeseries` / `stat` / `bargauge` / `gauge`) and the four metric names in the `expr` column.

#### References

- Grafana — build a dashboard and add panels: https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/
- Grafana visualizations (Stat, Bar gauge, Time series): https://grafana.com/docs/grafana/latest/panels-visualizations/visualizations/
- PromQL `histogram_quantile` — the p95 latency panel: https://prometheus.io/docs/prometheus/latest/querying/functions/#histogram_quantile
