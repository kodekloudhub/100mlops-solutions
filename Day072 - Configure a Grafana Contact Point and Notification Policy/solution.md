# Solution

An alert rule that fires but reaches nobody is useless. This task closes the **delivery** half of Grafana alerting: a **contact point** (a webhook to an in-stack receiver) that answers "where does a notification go?", and a **notification policy** route that answers "which alerts go there?" — matching `severity=high` and routing to that contact point. Together they turn a firing rule into an actual page.

> As an MLOps engineer, you wire alert *delivery* so a real signal reaches a human — the alert rule condition is only half the loop. The alert data is synthetic.

#### Follow the steps below

##### 1. Confirm the starting state.
From a VS Code terminal:
```
docker ps --format '{{.Names}}\t{{.Status}}'
docker exec webhook-sink python3 -c 'import urllib.request; print(urllib.request.urlopen("http://localhost:5000/health").read().decode())'
```
Four containers are running (`metric-emitter`, `webhook-sink`, `mon-prometheus`, `mon-grafana`). The webhook sink returns `{"status":"up"}`.

##### 2. Log in to Grafana.
Click the **Grafana** button at the top of the lab. Enter:
- **Username:** `admin`
- **Password:** `grafana2026`

##### 3. Create the webhook contact point.
From the left navigation:
- Click **Alerting** (bell icon) — the Alerting landing page shows tiles for Alert rules, Contact points, and Notification policies.
- Click **Manage contact points** on the Contact points tile (or use the sidebar's **Notification configuration** entry on newer builds).
- Click **+ Add contact point** (top right).
- **Name:** `mlops-oncall`.
- **Integration:** select **Webhook**.
- **URL:** `http://webhook-sink:5000/hook`.
- Leave the rest at defaults.
- (Optional) click **Test** — the sink container logs the test payload on stdout and appends it to `/var/log/sink/alerts.log`.
- Click **Save contact point**.

##### 4. Add a notification policy for `severity=high`.
From the left navigation:
- Click **Alerting** -> **Manage notification policies** (from the tiles), or sidebar **Notification configuration** → **Notification policies** tab on newer builds.
- On the **Default policy** row, click **+ Add route** (right-hand side of the row).
- In the **Add route** dialog:
  - **Matching labels:** add one row with **Label** `severity`, **Operator** `=`, **Value** `high`.
  - **Contact point:** `mlops-oncall`.
  - Leave the toggles (Continue matching, Override grouping/timings) off.
- Click **Add route** (blue button, bottom-right of the dialog).

The root policy now shows a nested child routing `severity=high` to the `mlops-oncall` contact point.

##### 5. Verify via Grafana's API.
From a VS Code terminal:
```
curl -s -u admin:grafana2026 http://localhost:3000/api/v1/provisioning/contact-points \
  | python3 -m json.tool
curl -s -u admin:grafana2026 http://localhost:3000/api/v1/provisioning/policies \
  | python3 -m json.tool
```
The first call shows the `mlops-oncall` contact point with `"type": "webhook"` and `"url": "http://webhook-sink:5000/hook"`. The second call shows the root policy with a nested route whose `object_matchers` contain `["severity", "=", "high"]` and whose `receiver` is `mlops-oncall`.

#### References

- Grafana — manage contact points (the webhook endpoint): https://grafana.com/docs/grafana/latest/alerting/configure-notifications/manage-contact-points/
- Grafana — notification policies (label-matched routing): https://grafana.com/docs/grafana/latest/alerting/configure-notifications/create-notification-policy/
