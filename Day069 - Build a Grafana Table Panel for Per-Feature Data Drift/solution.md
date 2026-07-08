# Solution

Some signals read better as a table than a chart. Per-feature data drift is one: you want each input feature on its own row with its latest drift score, scannable at a glance. This task builds a Grafana **table** panel over the `data_drift_score` metric, relying on Grafana rendering each Prometheus series' `column` label as its own row.
> As an MLOps engineer, you surface per-feature drift in a form an on-call engineer can scan in seconds — you are not computing drift here; the scores are synthetic.

#### Follow the steps below

##### 1. Confirm the starting state.
From a VS Code terminal:
```
docker ps --format '{{.Names}}\t{{.Status}}'
curl -s -u admin:grafana2026 http://localhost:3000/api/datasources | python3 -m json.tool
```
All three containers are running. The Prometheus datasource is pre-provisioned.

##### 2. Log in to Grafana.
Click the **Grafana** button at the top of the lab. Enter:
- **Username:** `admin`
- **Password:** `grafana2026`

##### 3. Open a new dashboard.
From the left navigation:
- Click **Dashboards** (four-squares icon).
- Click **New → New dashboard** (top right).
- Click **Add visualization** on the empty canvas.
- Select **Prometheus** when Grafana asks which data source to use.

##### 4. Query the drift-score metric.
In the query editor's **Code** tab, enter:
```
data_drift_score
```
Click **Run queries**. The default Time-series chart renders three lines (one per feature column — `amount`, `hour`, `num_tx_past_day`).

##### 5. Switch the visualization to Table.
On the right-hand sidebar, click the visualization selector (currently **Time series**). Choose **Table**.

The canvas re-renders as a table — each time series becomes a row, with the metric labels as columns and the most-recent value in the rightmost column. The three feature columns are visible as rows.

##### 6. Title the panel.
In the right sidebar, set:
- **Title:** `Drift score by column`

##### 7. Save the dashboard.
Click the **Save dashboard** icon (disk, top-right). Title the dashboard:
- **Dashboard title:** `Data drift`

Click **Save**.

##### 8. Verify via Grafana's API.
From a VS Code terminal:
```
curl -s -u admin:grafana2026 'http://localhost:3000/api/search?type=dash-db' | python3 -m json.tool
DASH_UID=$(curl -s -u admin:grafana2026 'http://localhost:3000/api/search?type=dash-db' \
  | python3 -c "import json, sys; print(json.load(sys.stdin)[0]['uid'])")
curl -s -u admin:grafana2026 http://localhost:3000/api/dashboards/uid/$DASH_UID \
  | python3 -c "import json, sys; d=json.load(sys.stdin); [print(p['type'], '|', p['targets'][0].get('expr')) for p in d['dashboard']['panels']]"
```
The detail output shows `table | data_drift_score` — the table visualization and the right query landed in the saved dashboard.

#### References

- Grafana — the Table visualization: https://grafana.com/docs/grafana/latest/panels-visualizations/visualizations/table/
- Grafana — build a dashboard and add a panel: https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/
- Prometheus querying — labels and instant vectors (the per-`column` series): https://prometheus.io/docs/prometheus/latest/querying/basics/
