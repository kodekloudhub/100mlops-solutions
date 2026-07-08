# Solution

A monitoring stack starts with wiring: Grafana can't chart anything until it knows where the metrics live. This task registers the running Prometheus server as a Grafana **data source** — the connection every panel and alert builds on — using Prometheus's in-network address (`http://prometheus:9090`, the compose service name, not `localhost`) and confirming the link with **Save & test**.

> As an MLOps engineer, you connect the dashboarding layer to the metrics store once, so every downstream panel queries a single source of truth — you are not analysing metrics yet, just plumbing the pipe. The metrics are synthetic.

#### Follow the steps below

##### 1. Confirm the starting state.
From a VS Code terminal:
```
docker ps --format '{{.Names}}\t{{.Status}}'
curl -s http://localhost:9090/-/ready
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:3000/login
```
All three containers (`metric-emitter`, `mon-prometheus`, `mon-grafana`) are running. Prometheus answers `Prometheus Server is Ready.`. Grafana's login page returns `200`.

##### 2. Open Grafana and log in.
Click the **Grafana** button at the top of the lab. On the login page enter:
- **Email or username:** `admin`
- **Password:** `grafana2026`

##### 3. Add Prometheus as a data source.
From the left navigation:
- Click the **Connections** icon (the plug) in the left-hand bar.
- Click **Data sources**.
- Click **+ Add new data source** (top right).
- Select **Prometheus** from the list.

The data-source configuration form opens.

##### 4. Fill in the connection details.
In the **Connection** section:
- **Prometheus server URL:** `http://prometheus:9090`

Leave every other field at its default value — no auth, no TLS, no custom headers. The Grafana container and the Prometheus container share the compose network, so the URL uses the compose service name `prometheus`, not `localhost`.

##### 5. Save & test.
Scroll to the bottom of the form and click **Save & test**. A green banner reading **"Data source is working"** appears next to a sample-data preview. Note the **UID** Grafana assigned to the data source — it's in the URL bar after the save.

##### 6. Verify the live connection.
From a VS Code terminal:
```
curl -s -u admin:grafana2026 http://localhost:3000/api/datasources | python3 -m json.tool
```
The response lists one data source with `"type": "prometheus"` and `"url": "http://prometheus:9090"`.

Confirm the datasource health check via Grafana's API:
```
DS_UID=$(curl -s -u admin:grafana2026 http://localhost:3000/api/datasources \
  | python3 -c "import json, sys; print(json.load(sys.stdin)[0]['uid'])")
curl -s -u admin:grafana2026 \
  http://localhost:3000/api/datasources/uid/$DS_UID/health | python3 -m json.tool
```
The response reads `"status": "OK"` — Grafana can reach Prometheus and query metrics through it.

##### 7. Build a first panel to confirm data flows.
Prove the datasource actually returns data by putting one metric on a dashboard:
- In the left navigation, click **Dashboards → New → New dashboard → + Add visualization**.
- Select the **Prometheus** data source.
- In the query editor's **Code** tab, enter `flask_http_request_total` and click **Run queries**. The panel renders the emitter's request counter climbing — proof Prometheus is scraping and Grafana can read it. This panel is just the "is data flowing" confirmation.
- Give the panel a title (e.g. `Request volume`), then click **Save dashboard** (top right), name it (e.g. `Monitoring smoke test`), and **Save**.

Verify from a VS Code terminal that the dashboard persisted with a panel:
```
curl -s -u admin:grafana2026 'http://localhost:3000/api/search?type=dash-db' | python3 -m json.tool
```
The response lists the dashboard you saved — the Grafana → Prometheus pipeline is live end to end.

#### References

- Grafana — the Prometheus data source: https://grafana.com/docs/grafana/latest/datasources/prometheus/
- Grafana — build a dashboard and add a panel: https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/
- Prometheus querying basics (PromQL): https://prometheus.io/docs/prometheus/latest/querying/basics/
