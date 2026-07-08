# Solution

**The platform.** xFusionCorp runs `fraud-detector` as a small production ML system: models are trained and served behind the `production` registry alias, an automated loop retrains on drift, the server is deployed by GitOps, and the running service is observed with Prometheus + Grafana. **This task builds the GitOps deployment layer** — the git repo is the source of truth, and an ArgoCD Application reconciles the cluster against it.

You wire and drive the loop: author the Application so ArgoCD tracks the `mlops-deploy` manifests, apply and sync it to deploy the server, then roll a new image version by bumping the tag in git and syncing again — so rolling out a new version is a committed change in git, not a hand-edit of the running workload. (You apply the Application resource once to bootstrap the reconciler; from then on the server itself is deployed and updated through git.)

> As an MLOps engineer, you wire a git-driven reconciler and roll a model-server version through it, so the workload is deployed from a reviewable git commit rather than changed by hand on the cluster — you are not building or evaluating the model; the server is an `nginx` stand-in.

#### Follow the steps below

##### 1. Confirm the pre-staged environment.
From a VS Code terminal:
```
curl -s -o /dev/null -w 'gitea=%{http_code}\n' http://localhost:3000/api/v1/version
curl -s -o /dev/null -w 'argocd=%{http_code}\n' http://localhost:5000/
kubectl get applications -n argocd
```
Gitea and ArgoCD answer `200`. No `fraud-detector` Application exists yet, and nothing is deployed in `default` — you create all of that below.

##### 2. Author and apply the ArgoCD Application.
`/root/code/application.yaml` is a scaffold with three `TODO`s. Fill them so the Application tracks the `mlops-deploy` repo:
```yaml
  source:
    repoURL: http://gitea-http.gitea.svc.cluster.local:3000/gitops-admin/mlops-deploy.git
    targetRevision: HEAD
    path: manifests
  destination:
    server: https://kubernetes.default.svc
    namespace: default
```
Apply it:
```
kubectl apply -n argocd -f /root/code/application.yaml
```
The `fraud-detector` Application now appears in the ArgoCD UI, initially `OutOfSync` / `Missing` (the manifests haven't been applied to the cluster yet).

##### 3. Sync to deploy the server.
Click the **ArgoCD UI** button. Log in with `admin` / `adminadmin`. Click into the `fraud-detector` Application tile, then click **SYNC → SYNCHRONIZE**. ArgoCD applies `manifests/` from the repo: the `fraud-detector` Deployment (`nginx:1.25-alpine`) and its NodePort Service are created, and the Application flips to `Synced` + `Healthy`. Verify:
```
kubectl get deploy fraud-detector -n default \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
curl -s -o /dev/null -w 'app=%{http_code}\n' http://localhost:8085/
```
The image is `nginx:1.25-alpine` and the host NodePort returns `200`.

##### 4. Bump the image tag in Gitea.
Click the **Gitea UI** button. Log in with `gitops-admin` / `adminadmin`. Open `gitops-admin/mlops-deploy → manifests/deployment.yaml`. Click the pencil **Edit file** icon (top-right of the file viewer).

Change:
```yaml
          image: nginx:1.25-alpine
```
to:
```yaml
          image: nginx:1.27-alpine
```
Ensure **Commit directly to the `main` branch** is selected, then click **Commit changes**.

##### 5. Refresh the ArgoCD Application.
Back in the **ArgoCD UI**, on the `fraud-detector` Application click **REFRESH**. ArgoCD rescans the repo at HEAD; the badge flips to `OutOfSync` because the cluster's image tag no longer matches the desired state. The **DIFF** tab shows a single change: the container image tag.

##### 6. Sync the Application.
Click **SYNC → SYNCHRONIZE** (leave **PRUNE** unchecked). ArgoCD applies the new manifest; the old pod terminates and a new pod starts on `nginx:1.27-alpine`. The Application returns to `Synced` + `Healthy`.

##### 7. Verify the rollout.
```
kubectl get deploy fraud-detector -n default \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
kubectl get pods -n default -l app=fraud-detector
curl -s -o /dev/null -w 'app=%{http_code}\n' http://localhost:8085/
```
The image is now `nginx:1.27-alpine`, the new pod is `Running`/`1/1`, and the host NodePort still returns `200`.

##### 8. Confirm via the ArgoCD REST API.
```
TOKEN=$(curl -s -X POST http://localhost:5000/api/v1/session \
  -H 'Content-Type: application/json' \
  -d '{"username":"admin","password":"adminadmin"}' \
  | python3 -c "import json, sys; print(json.load(sys.stdin)['token'])")

curl -s -H "Authorization: Bearer $TOKEN" \
  http://localhost:5000/api/v1/applications/fraud-detector \
  | python3 -c "
import json, sys
app = json.load(sys.stdin)
status = app.get('status', {})
print('sync   :', status.get('sync', {}).get('status'))
print('health :', status.get('health', {}).get('status'))
"
```
Output: `sync: Synced`, `health: Healthy`.

#### References

- Argo CD — declarative Applications (repo / path / destination reconciliation): https://argo-cd.readthedocs.io/en/stable/user-guide/application-specification/
- Argo CD — getting started (create, Refresh, Sync): https://argo-cd.readthedocs.io/en/stable/getting_started/
