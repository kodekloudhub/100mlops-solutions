# Solution

A monitoring stack is only as useful as its weakest link: Evidently can score drift perfectly and still page nobody if the chain from metric → scrape → datasource → dashboard is broken anywhere along the way. This capstone hands you an end-to-end stack (Evidently drift scorer → Flask metric-emitter → Prometheus → Grafana) with three silent wiring bugs — none crashes a container, so you diagnose by reading the *symptoms* (a 404 `/metrics`, a DOWN Prometheus target, an empty Grafana panel) back to the one config file each lives in — then build a tagged overview dashboard on top of the repaired pipeline.

> As an MLOps engineer, you debug observability the way you debug production: start from the symptom, trace the signal to the first broken hop, and fix the config — the drift scores were fine all along; the plumbing wasn't. The metrics are synthetic.

#### Follow the steps below

##### 1. Read the failure symptoms first.
From a VS Code terminal:
```
docker ps --format '{{.Names}}\t{{.Status}}'
cat /root/code/monitoring/drift/drift_scores.json
docker exec metric-emitter python3 -c 'import urllib.request; print(urllib.request.urlopen("http://localhost:5000/metrics").status)'
curl -s http://localhost:9090/api/v1/targets | python3 -m json.tool | head -40
curl -s -u admin:grafana2026 http://localhost:3000/api/datasources | python3 -m json.tool
```
All three containers are `Up`, and the **Evidently drift scorer is doing its job** — `drift_scores.json` carries fresh per-column PSI values. Confirm it visually too: click the **Evidently UI** button (port `8000`) → `fraud-detector drift monitoring` → the **Dashboard** tab plots drift share and per-column PSI accumulating every ~minute, and the **Reports** tab lists the underlying runs. So the source is healthy — but the emitter's `/metrics` returns `404`, Prometheus's target is DOWN, and Grafana's datasource URL ends in `:9091`. The Evidently scores stop dead at the first broken link: three independent bugs, one per config file, and none of them in the scorer.

##### 2. Bug 1 — emitter route.
Open `/root/code/monitoring/app/metric_emitter.py`. Near the bottom, the `@app.route("/prom-metrics")` decorator is wrong — Prometheus scrapes `/metrics` by default. Change it:
```python
@app.route("/metrics")
def metrics():
    return generate_latest(REGISTRY), 200, {"Content-Type": CONTENT_TYPE_LATEST}
```
Save.

##### 3. Bug 2 — Prometheus scrape target port.
Open `/root/code/monitoring/prometheus.yml`. The target port is wrong — the emitter listens on `:5000`, not `:8000`:
```yaml
scrape_configs:
  - job_name: metric-emitter
    static_configs:
      - targets:
          - metric-emitter:5000
```
Save.

##### 4. Bug 3 — Grafana datasource URL.
Open `/root/code/monitoring/grafana/provisioning/datasources/prometheus.yml`. Prometheus serves on `:9090`, not `:9091`:
```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: true
```
Save.

##### 5. Reload the affected services.
```
cd /root/code/monitoring
docker compose restart metric-emitter prometheus grafana
```

Give the restart ~5 s, then re-check:
```
docker exec metric-emitter python3 -c 'import urllib.request; print(urllib.request.urlopen("http://localhost:5000/metrics").status)'
curl -s http://localhost:9090/api/v1/targets \
  | python3 -c "import json, sys; [print(t['labels']['job'], t['health']) for t in json.load(sys.stdin)['data']['activeTargets']]"
curl -s -u admin:grafana2026 http://localhost:3000/api/datasources \
  | python3 -c "import json, sys; [print(d['name'], d['url']) for d in json.load(sys.stdin)]"
```
`/metrics` prints `200`, the Prometheus target is `up`, and the Grafana datasource URL is `http://prometheus:9090`. If the target still reads `down`, Prometheus simply hasn't taken its next scrape yet — wait 5-10 s and re-run the targets check. The full chain is restored: Evidently scorer -> emitter -> Prometheus -> Grafana. Spot-check the Evidently signal end-to-end with `curl -s 'http://localhost:9090/api/v1/query?query=data_drift_score' | python3 -m json.tool | head -30`.

##### 6. Cross-check in the Evidently UI.
Click the **Evidently UI** button at the top of the lab (port `8000`):
- Click the `fraud-detector drift monitoring` project.
- The **Dashboard** tab (default) renders two pre-configured line charts — **Share of drifted columns** and **Per-column drift (PSI)** — one point per scoring run, accumulating every ~minute since the lab booted. This half of the stack never broke: the points kept landing even while Grafana was blind.
- The **Reports** tab lists the underlying runs; **View** any row → one card per feature renders, headed e.g. `Drift in column 'amount' — Data drift detected. Drift detection method: PSI. Drift score: 0.627`. Each card has two tabs: **Data Drift** (current mean ± std against the reference band) and **Data Distribution** (current-vs-reference histogram).

These are the same PSI values your fixed pipeline now carries into Grafana as `data_drift_score`.

##### 7. Log in to Grafana.
Click the **Grafana** button at the top of the lab. Enter:
- **Username:** `admin`
- **Password:** `grafana2026`

##### 8. Build the tagged monitoring-overview dashboard.
From the left navigation:
- Click **Dashboards -> New -> New dashboard**.

Panel 1:
- Click **Add visualization** -> **Prometheus**.
- **Code** tab: `sum(rate(flask_http_request_total[1m])) by (version)`.
- Leave visualization as **Time series**.
- Title: `Request rate`.
- **Back to dashboard**.

Panel 2:
- **Add -> Visualization** -> **Prometheus**.
- **Code** tab: `histogram_quantile(0.95, sum(rate(model_inference_duration_seconds_bucket[5m])) by (le))`.
- Title: `p95 inference latency (s)`.
- **Back to dashboard**.

Panel 3:
- **Add -> Visualization** -> **Prometheus**.
- **Code** tab: `prediction_accuracy`.
- Visualization: **Stat**.
- Title: `Prediction accuracy`.
- **Back to dashboard**.

##### 9. Save, then add a tag.
Grafana 13 splits title+description into the first Save dialog and moves tags into the dashboard settings. Save in two passes:

1. Click the **Save dashboard** icon (disk, top-right). In the drawer, set **Title** to `Monitoring overview` and click **Save**.
2. Open **Dashboard options** (top-right) → **Settings** (or the gear icon on the toolbar) → **General** tab.
3. In the **Tags** field, type `mlops` and press Enter. Optionally add more (e.g. `monitoring`, `fraud`) the same way.
4. Click **Save dashboard** inside the settings pane to persist the tag change.

The dashboards list now shows the dashboard with a coloured tag pill.

##### 10. Verify end-to-end.
From a VS Code terminal:
```
DASH_UID=$(curl -s -u admin:grafana2026 'http://localhost:3000/api/search?type=dash-db' \
  | python3 -c "import json, sys; print(json.load(sys.stdin)[0]['uid'])")
curl -s -u admin:grafana2026 http://localhost:3000/api/dashboards/uid/$DASH_UID \
  | python3 -c "
import json, sys
d = json.load(sys.stdin)['dashboard']
print('title:', d.get('title'))
print('tags: ', d.get('tags'))
for p in d.get('panels', []):
    expr = (p['targets'][0].get('expr') if p.get('targets') else '-')
    print(f\"{p['type']:12s} | {p['title']:30s} | {expr}\")
"
```
The output prints the dashboard title, a non-empty tags list, and three rows — one per panel — each with its query and visualization type.

#### References

- Prometheus scrape configuration — the `targets` port (Bug 2): https://prometheus.io/docs/prometheus/latest/configuration/configuration/#scrape_config
- Grafana datasource provisioning — the datasource URL (Bug 3): https://grafana.com/docs/grafana/latest/administration/provisioning/
- Grafana — build a dashboard and tag it: https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/
