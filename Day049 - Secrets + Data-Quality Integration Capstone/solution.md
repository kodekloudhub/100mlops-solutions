# Solution

This capstone composes three earlier sections into one CI release pipeline on the `fraud-detector` repo. A **Gitea Actions** workflow, triggered by a pull request, runs ordered jobs: **fetch-secret** pulls the MLflow credential from **Vault** (no hardcoded secret), **data-quality** runs a **Great Expectations** checkpoint as a gate, and — only if both pass — the model is trained and registered in **MLflow**. The skill is wiring these together so the PR is a single gated release. This task authors the Vault-read step in the workflow, stages the secret, opens and merges the PR to run the pipeline, and promotes the registered model to the `production` alias.

> As an MLOps engineer, you compose secrets management and a data-quality gate into one CI release pipeline—you are not judging model or data quality scientifically. The data and model are synthetic.

#### Follow the steps below

##### 1. Stage the MLflow credential in Vault.
Click the **Vault** button at the top of the lab. From a VS Code terminal:
```
cat /root/code/vault-token
```
Copy the token and paste it into the Vault **Token** login field.

Inside the Vault UI:
- Navigate to **Secrets** → `secret/` (KV v2 engine, pre-enabled).
- Click **Create secret**.
  - **Path for this secret:** `mlflow`.
  - Add one key/value pair: **key** `mlflow_password`, **value** any non-empty string (for example `prod-release-2026`).
  - Click **Save** **once**. (If you click Save again you'll see `check-and-set parameter did not match the current version` — that just means the secret already saved on the first click; don't re-Save, the credential is already staged.)

  If you prefer the terminal, the same thing without the UI's create-only check is:
  ```
  export VAULT_ADDR=http://127.0.0.1:8200 VAULT_TOKEN=$(cat /root/code/vault-token)
  vault kv put secret/mlflow mlflow_password=prod-release-2026
  ```

##### 2. Author the `fetch-secret` Vault-read step.
Open `/root/code/fraud-detector/.gitea/workflows/production.yml` in the VS Code editor. The `fetch-secret` job's read step is a `# TODO`. Replace the two placeholder lines (`echo ...` / `exit 1`) with a read of the Vault KV entry that fails the job when the key is absent:
```yaml
      - name: Read MLflow password from Vault
        run: |
          TOKEN=$(cat /root/code/vault-token)
          PASSWORD=$(curl -sf -H "X-Vault-Token: $TOKEN" \
            "$VAULT_ADDR/v1/secret/data/mlflow" \
            | python3 -c "import json, sys; print(json.load(sys.stdin)['data']['data']['mlflow_password'])")
          if [ -z "$PASSWORD" ]; then
            echo "::error::Empty password from Vault -- stage mlflow_password in secret/mlflow first"
            exit 1
          fi
          echo "::notice::Fetched MLflow password from Vault (len=${#PASSWORD})"
```
Commit and push to the feature branch:
```
cd /root/code/fraud-detector
git add .gitea/workflows/production.yml
git commit -m "ci: read MLflow credential from Vault in fetch-secret job"
git push
```

##### 3. Open the pull request in Gitea.
Click the **Gitea** button. Log in with `gitea-admin` / `gitea2026`. Open the `fraud-detector` repo.

- Click **Pull Requests** (top nav) → **+ New Pull Request**.
- **Base:** `main`.
- **Compare (head):** `production-release`.
- **Title:** `Cut production release` (or similar).
- Click **Create Pull Request**.

Opening the PR triggers the `Production release` workflow.

##### 4. Watch the three jobs go green.
Open the repo's **Actions** tab (top nav) and click the latest run (Gitea shows workflow runs under **Actions**, not a PR "Checks" tab). The workflow runs three jobs in sequence (each `needs:` the previous):

- **`fetch-secret`** → reads `secret/mlflow.mlflow_password` from Vault. Log: `::notice::Fetched MLflow password from Vault (len=...)`.
- **`data-quality`** → runs the `schema_check` Great Expectations checkpoint against `data/transactions.csv`. Log ends with `Checkpoint schema_check success=True`.
- **`register-model`** → logs a run to MLflow and registers it as `fraud-detector`. Log: `Registered model versions: ['1']`.

Total runtime: ~60–90 s.

##### 5. Merge the pull request.
Back on the PR page, click **Merge pull request** → **Create merge commit** → **Merge**. The PR is now merged into `main`.

##### 6. Promote the model in MLflow.
Click the **MLflow UI** button. Navigate to **Models** → `fraud-detector`. Click **Version 1** (the version the workflow just registered). On the version page, find the **Aliases** panel:

- Click **+ Add alias**.
- Enter `production`.
- Click **Save aliases**.

##### 7. Verify end-to-end.
```
# Vault has mlflow_password
VAULT_TOKEN=$(cat /root/code/vault-token)
curl -s -H "X-Vault-Token: $VAULT_TOKEN" \
  http://localhost:8200/v1/secret/data/mlflow \
  | python3 -c "import json, sys; d=json.load(sys.stdin)['data']['data']; print('keys:', list(d.keys()))"

# PR is merged + workflow ran green
TOKEN=$(cat /root/.gitea/token)
curl -s -H "Authorization: token $TOKEN" \
  'http://localhost:3000/api/v1/repos/gitea-admin/fraud-detector/pulls?state=all&sort=newest' \
  | python3 -c "
import json, sys
pr = next(p for p in json.load(sys.stdin) if p['head']['ref']=='production-release')
print('merged :', pr['merged'])
print('sha    :', pr['head']['sha'][:8])
"

# fraud-detector has production alias
curl -s 'http://localhost:5000/api/2.0/mlflow/registered-models/alias?name=fraud-detector&alias=production' \
  | python3 -m json.tool
```

The three calls print, respectively: a keys list containing `mlflow_password`; `merged: True`; and a JSON body whose `model_version.version` is a non-zero integer.

> Every UI exists because a different role owns its piece: Vault owns the secret, Gitea owns the review, MLflow owns the promotion. A real release tour visits all three; this capstone asks you to do exactly that.

#### References

- Vault KV v2 HTTP API — the `secret/data/<path>` read the `fetch-secret` job performs: https://developer.hashicorp.com/vault/api-docs/secret/kv/kv-v2
- Gitea Actions overview — how the runner executes `.gitea/workflows/`: https://docs.gitea.com/usage/actions/overview
