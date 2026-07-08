# Solution

GitOps is declarative deployment with a git repo as the single source of truth: ArgoCD's controller loop continuously reconciles the live cluster to match the manifests in the repo. This task has you create an ArgoCD Application through the UI pointing at the canonical guestbook example, enable automatic sync with prune and self-heal, and confirm the cluster converges to `Synced` + `Healthy`.

> As an MLOps engineer, you deliver workloads via GitOps so the cluster state always matches git — there is no model here; the guestbook app is a model-agnostic stand-in for the reconcile loop you are practising.

#### Follow the steps below

##### 1. Confirm ArgoCD is up.
From a VS Code terminal:
```
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:5000/
kubectl -n argocd get deploy
```
The UI answers `200`; all argocd-* deployments are Available.

##### 2. Log in to the ArgoCD UI.
Click the **ArgoCD UI** button. Credentials:
- **Username:** `admin`
- **Password:** `admin`

The landing page shows the Applications dashboard — empty on first open.

##### 3. Create the Application.
Click **+ NEW APP** (top-left). Fill the form:

| Field | Value |
|---|---|
| Application Name | `guestbook` |
| Project Name | `default` |
| Sync Policy | `Automatic` with **PRUNE RESOURCES** + **SELF HEAL** both ticked |
| Repository URL | `https://github.com/argoproj/argocd-example-apps` |
| Revision | `HEAD` |
| Path | `guestbook` |
| Cluster URL | `https://kubernetes.default.svc` |
| Namespace | `default` |

Click **CREATE** (top-left of the form).

##### 4. Watch it sync.
The Applications dashboard shows the `guestbook` tile. Because Sync Policy is `Automatic`, ArgoCD pulls the repo and applies the manifests without a manual Sync click. Within ~30-60 s:
- Tile header flips from `OutOfSync` to `Synced`.
- Health badge flips from `Missing`/`Progressing` to `Healthy`.

Click the tile; the resource tree shows `guestbook-ui` Deployment + Service as green nodes.

##### 5. Verify on the cluster side.
```
kubectl get deploy -n default
kubectl get svc -n default
kubectl get application guestbook -n argocd -o jsonpath='{.status.sync.status}/{.status.health.status}{"\n"}'
```
- `guestbook-ui` Deployment + Service exist in `default`.
- The `jsonpath` prints `Synced/Healthy`.

##### 6. Verify via the ArgoCD REST API.
```
TOKEN=$(curl -s -X POST http://localhost:5000/api/v1/session \
  -d '{"username":"admin","password":"admin"}' \
  -H 'Content-Type: application/json' \
  | python3 -c "import json, sys; print(json.load(sys.stdin)['token'])")

curl -s -H "Authorization: Bearer $TOKEN" \
  http://localhost:5000/api/v1/applications/guestbook \
  | python3 -c "
import json, sys
app = json.load(sys.stdin)
spec = app.get('spec', {}).get('source', {})
status = app.get('status', {})
print('repoURL   :', spec.get('repoURL'))
print('path      :', spec.get('path'))
print('sync      :', status.get('sync', {}).get('status'))
print('health    :', status.get('health', {}).get('status'))
"
```
Output confirms repoURL, path, `sync=Synced`, `health=Healthy`.

#### References

- Argo CD — getting started (creating an Application from a git repo): https://argo-cd.readthedocs.io/en/stable/getting_started/
- Argo CD — automated sync policy (auto-sync, prune, self-heal): https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/
