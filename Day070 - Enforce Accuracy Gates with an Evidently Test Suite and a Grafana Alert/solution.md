# Solution

A quality threshold is only useful if it is enforced. This task enforces one accuracy bar at **two altitudes**: an **Evidently** test suite that fails a pipeline before a degraded model ships, and a **Grafana alert rule** that pages on-call when live accuracy slips past the same bar. You complete the Evidently test-suite scaffold (thresholded metrics via the current `Report(..., include_tests=True)` API) and wire the matching Grafana alert.

> As an MLOps engineer, you codify a quality bar as automated machinery — enforced pre-deploy by the test suite and at runtime by the alert — so a bad model is caught by tooling, not by a customer. The metrics are synthetic.

#### Follow the steps below

##### 1. Confirm the starting state.
From a VS Code terminal:
```
docker ps --format '{{.Names}}\t{{.Status}}'
curl -s -u admin:grafana2026 http://localhost:3000/api/datasources | python3 -m json.tool
curl -s http://localhost:9090/api/v1/query?query=prediction_accuracy | python3 -m json.tool | head -30
head -5 /root/code/monitoring/tests/current.csv
```
All three containers are running. The Prometheus datasource is pre-provisioned. Prometheus is already returning a `prediction_accuracy` value in the 0.70 - 0.95 band. The production batch CSV carries the feature columns plus `is_fraud` (target) and `prediction` (the model's output). The test-suite scaffold sits next to it at `tests/test_suite.py`, and the **Evidently UI** button (port `8000`) opens the `fraud-detector quality gates` project — empty until you publish a run.

##### 2. Complete the test-suite scaffold.
Open `/root/code/monitoring/tests/test_suite.py` in the VS Code editor. The scaffold is Evidently's "monitoring as tests" mode with everything pre-wired — data loading, the classification mapping, `include_tests=True`, the results dump, and publishing to the Evidently UI. Only the gates are missing. Replace the `# (append the two thresholded metrics here)` line below the TODO block with:
```python
METRICS.append(DatasetMissingValueCount(tests=[lt(10)]))
METRICS.append(Accuracy(tests=[gt(0.80)]))
```
`tests=[lt(10)]` means *"fail if missing values >= 10"*, and `tests=[gt(0.80)]` means *"fail if accuracy <= 0.80"* — assertion conditions attached directly to metrics, perfect for CI gates that should block a deploy if data or model quality degrades. Save.

##### 3. Run the suite.
```
python3 /root/code/monitoring/tests/test_suite.py
```
Expected output: `Tests: 2/2 passed` — the batch carries 3 missing values (under the 10 gate) and ~0.89 accuracy (above the 0.80 gate). The run also publishes itself to the Evidently workspace.

##### 4. Inspect the verdict in the Evidently UI.
Click the **Evidently UI** button at the top of the lab (port `8000`):
- The home page lists the `fraud-detector quality gates` project — click it.
- Open the **Reports** tab. One row per `python3 test_suite.py` execution appears — click **View** on your run.
- The **Metrics** tab renders the raw numbers: missing-values `count 3.0` / `share 0.01` and `Accuracy metric 0.890`.
- The **Tests** tab renders the verdicts: a `2 SUCCESS / 0 WARNING / 0 FAIL / 0 ERROR` summary with a green badge per gate — `Actual value 3.000 < 10.000` and `Actual value 0.890 >= 0.800`.

This is the same JSON you dumped to `test_results.json`, rendered for a reviewer who never touches code. Optionally double-check the file too:
```
python3 -m json.tool /root/code/monitoring/tests/test_results.json | head -40
```

##### 5. Log in to Grafana.
Click the **Grafana** button at the top of the lab. Enter:
- **Username:** `admin`
- **Password:** `grafana2026`

##### 6. Open the alert-rule form.
From the left navigation:
- Click **Alerting** (bell icon).
- Click **Alert rules**.
- Click **New alert rule** (top right).

At the top of the form, in the **Enter a name** field, type `Prediction accuracy below 0.80`.

##### 7. Build the query.
In the **Define query and alert condition** section:
- **Data source:** `Prometheus`.
- **Code** tab, expression:
  ```
  avg_over_time(prediction_accuracy[1m])
  ```
- Click **Run queries**. The preview **Table** renders the latest value (roughly 0.85 under healthy conditions).

##### 8. Set the threshold.
Grafana 13 inlines the threshold directly under the query as an **Alert condition** row. On that row:
- Change the operator dropdown from `IS ABOVE` to **`IS BELOW`**.
- Change the numeric field from `0` to **`0.80`**.
- Click **Preview alert rule condition** — the row reports `Firing` when the reduced value is below `0.80` and `Normal` otherwise.

##### 9. Pick (or create) the folder.
In the **Add folder and labels** section, set **Folder** to `MLOps` (click **+ New folder** if it doesn't exist yet). Leave **Labels** empty.

##### 10. Set the evaluation behavior.
In the **Set evaluation behavior** section:
- **Evaluation group and interval:** click **+ New evaluation group**, name it `model-health`, and set the interval to `1m`. Back on the form, pick `model-health` from the dropdown.
- **Pending period:** `1m` (the default — the rule must stay firing for this long before it pages).
- **Keep firing for:** leave at `None`.

##### 11. Pick a contact point.
In the **Configure notifications** section, the **Contact point** dropdown is required. Pick the pre-provisioned `default` entry (a no-op email receiver; routing an alert to a real webhook is a separate configuration step).

##### 12. (Optional) Add a notification message.
In the **Configure notification message** section, you can fill in an optional `Summary` like `Fraud-detection model rolling accuracy fell below threshold.` and a `Description`.

##### 13. Save the rule.
Scroll to the bottom of the form and click **Save** (blue button next to **Cancel**).

##### 14. Verify via Grafana's API.
From a VS Code terminal:
```
curl -s -u admin:grafana2026 http://localhost:3000/api/v1/provisioning/alert-rules \
  | python3 -m json.tool | head -80
```
The response includes the new rule, with `data[].model.expr` showing `avg_over_time(prediction_accuracy[1m])` and a threshold condition whose `evaluator.params` carries `0.80`. That is the exact same backing store the test suite inspects.

> **Why both?** The Evidently test suite and the Grafana alert enforce the *same* 0.80 accuracy gate at two altitudes: the suite fails a CI pipeline before a degraded model ships; the alert pages on-call after live accuracy slips. One threshold, two enforcement points.

#### References

- Evidently — tests and presets (`include_tests=True`, `lt`/`gt` conditions): https://docs.evidentlyai.com/
- Grafana — create an alert rule: https://grafana.com/docs/grafana/latest/alerting/alerting-rules/create-grafana-managed-rule/
- PromQL `avg_over_time` — smoothing the accuracy signal: https://prometheus.io/docs/prometheus/latest/querying/functions/#aggregation_over_time
