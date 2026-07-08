# Solution

A production serving stack is more than the model container: it sits behind a reverse proxy, is rate-limited, exports metrics, and is observable on a dashboard. This capstone brings up that full stack with `docker compose` — a Flask fraud API behind nginx, with Redis rate-limiting, Prometheus scraping, and Grafana — fixing the port-wiring bugs that stop it and building a Grafana panel for the request-rate metric.

> As an MLOps engineer, you assemble serving + proxy + metrics + dashboards into one reproducible stack so the service is operable, not just reachable — you are not evaluating the model. The data is synthetic.

#### Follow the steps below

##### 1. Confirm the starting state.
From a VS Code terminal:
```
cd /root/code/serving/production
ls
cat app/app.py | head -30
cat prometheus.yml
cat nginx.conf
```
Every scaffold file is in place. `app/app.py` imports `PrometheusMetrics` but never calls it. `prometheus.yml` scrapes `model-api:8000`. `nginx.conf` proxies to `model-api:8000`.

##### 2. Fix Bug 1 — instrument the Flask app.
Open `/root/code/serving/production/app/app.py` in the VS Code editor. After the `app = Flask(__name__)` line, add:
```python
metrics = PrometheusMetrics(app)
```
Save. The exporter attaches a `/metrics` route to the Flask app when it is instantiated with the app as its argument.

##### 3. Fix Bug 2 — Prometheus scrape target.
Open `/root/code/serving/production/prometheus.yml`. Change the target port:
```yaml
    static_configs:
      - targets:
          - model-api:5000
```
Save.

##### 4. Fix Bug 3 — nginx upstream.
Open `/root/code/serving/production/nginx.conf`. Change the upstream server line:
```nginx
    upstream model_backend {
        server model-api:5000;
    }
```
Save.

##### 5. Bring the stack up.
```
cd /root/code/serving/production
docker compose up -d
docker compose ps
```
Six containers report `Up` — the five observability-stack services (`model-api`, `prod-redis`, `prod-nginx`, `prod-prometheus`, `prod-grafana`) plus a `prod-traffic` sidecar that continuously POSTs to `/predict` through nginx so Grafana always has live request-rate data to plot. `model-api` additionally shows `(healthy)` once its container healthcheck passes, and every service carries a `restart: unless-stopped` policy — the production-shaping the compose already wires for you.

Confirm the observability loop:
```
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:5000/metrics
curl -s -X POST -H 'Content-Type: application/json' \
  -d '{"amount":3200,"hour":23,"num_tx_past_day":5}' \
  http://localhost:8085/predict
curl -s http://localhost:9090/api/v1/targets | python3 -m json.tool | head -30
```
`/metrics` returns `200`. The nginx-proxied `/predict` returns `{"is_fraud": ...}`. The Prometheus targets endpoint reports the `model-api` job with `"health": "up"`.

##### 6. Create the Grafana dashboard via the UI.
Click the **Grafana** button at the top of the lab. Log in with:
- **Username:** `admin`
- **Password:** `grafana2026`

From the left navigation:
1. Click **Dashboards** → **New** → **New dashboard**.
2. Click **Add visualization** on the empty canvas.
3. Select the pre-provisioned **Prometheus** datasource.
4. In the query editor's **Code** tab, enter:
   ```
   rate(flask_http_request_total[1m])
   ```
5. Click **Run queries**. The panel renders per-second request-rate lines split by status / path / method labels.
6. On the right sidebar, set **Title** to something along the lines of `Model API request rate` and **Visualization** to **Time series**.
7. Click **Save dashboard** (the disk icon at the top). Give the dashboard a title (e.g. `Fraud API production`) and click **Save**.

##### 7. Verify from the terminal.
```
curl -s -u admin:grafana2026 http://localhost:3000/api/datasources | python3 -m json.tool
curl -s -u admin:grafana2026 'http://localhost:3000/api/search?type=dash-db' | python3 -m json.tool
```
The first call lists the `Prometheus` datasource. The second call returns at least one row — the dashboard you saved. Open **Prometheus** (port 9090) in a second browser tab to confirm the scrape targets turn green in its **Status → Targets** page.

#### References

- `prometheus-flask-exporter` — instrumenting the Flask app with `PrometheusMetrics(app)`: https://github.com/rycus86/prometheus_flask_exporter
- Prometheus scrape configuration (`scrape_configs` / `targets`): https://prometheus.io/docs/prometheus/latest/configuration/configuration/#scrape_config
- Grafana — building a dashboard and panel: https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/
- nginx reverse proxy (`upstream` / `proxy_pass`): https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/
