# Solution

A secrets store is only as safe as its access rules. Vault is default-deny: a token can read nothing unless a **policy** grants it, and machines authenticate with **AppRole** rather than a static token. This task writes a least-privilege KV policy for the MLflow wrapper and fixes its AppRole wiring so it can read exactly its own secret — and nothing else.

> As an MLOps engineer, you grant machines least-privilege, short-lived access to secrets through scoped policies and AppRole auth, instead of handing out static tokens — there is no model work here.

#### Follow the steps below

##### 1. Confirm the broken starting state.
From a VS Code terminal:
```
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:5000/
tail -5 /var/log/mlflow-wrapper.log
```
MLflow returns `000` (connection refused) — it is not running. The wrapper log's last line reads `Waiting: AppRole role 'mlflow' not found yet ...`: the wrapper authenticates via AppRole, but the `mlflow` role does not exist yet, so it cannot even attempt a login.

##### 2. Log in to Vault with the ROOT token.
Click the **Vault** button at the top of the lab. On the login page:
- **Method:** Token
- **Token:** paste the value of `/root/code/vault-root-token`.

##### 3. Open the broken policy and spot both faults.
In the left navigation, open **Access control → ACL policies** → click the `mlflow-reader` row. The current rules read:
```hcl
path "secret/mlflow" {
  capabilities = ["create", "update"]
}
```
Two things are wrong. First the **path**: KV v2 serves secret *data* under the `secret/data/` infix, so a rule on `secret/mlflow` never matches a `GET /v1/secret/data/mlflow`. Second the **capabilities**: `read` is missing. Both would deny the wrapper's read.

##### 4. Rewrite the rule for the KV v2 data path with `read`.
Click **Edit policy**. Replace the block so it names the `data/` path and grants `read`:
```hcl
path "secret/data/mlflow" {
  capabilities = ["read"]
}
```
Click **Save**. (`create`/`update` are not needed — the wrapper only reads its secret.)

##### 5. Enable AppRole and create the `mlflow` role.
The wrapper authenticates with AppRole rather than a static token, so the auth method and role must exist. AppRole is set up from the terminal (its UI support is limited). Point the CLI at Vault with the root token, then enable the method and create a role bound to the fixed policy:
```
export VAULT_ADDR=http://127.0.0.1:8200
export VAULT_TOKEN=$(cat /root/code/vault-root-token)

vault auth enable approle

vault write auth/approle/role/mlflow \
  token_policies=mlflow-reader \
  token_ttl=1h token_max_ttl=4h
```
The wrapper self-bootstraps the role's `role_id`/`secret_id` and logs in — no credential files to write by hand.

##### 6. Watch MLflow come up.
Wait ~5 s (one wrapper cycle), then click the **MLflow UI** button, or from a terminal:
```
for i in {1..15}; do
  code=$(curl -s -o /dev/null -w '%{http_code}' http://localhost:5000/)
  echo "attempt $i: $code"
  [ "$code" = "200" ] && break
  sleep 3
done
tail -3 /var/log/mlflow-wrapper.log
```
The wrapper's last line reads `AppRole login OK and secret readable -- launching MLflow.` and the HTTP probe returns `200`.

##### 7. Verify via the Vault REST API.
```
export VAULT_ADDR=http://127.0.0.1:8200
ROOT=$(cat /root/code/vault-root-token)

# Policy now grants read on the KV v2 data path:
curl -s -H "X-Vault-Token: $ROOT" \
  "$VAULT_ADDR/v1/sys/policies/acl/mlflow-reader" \
  | python3 -c "import json,sys; print(json.load(sys.stdin)['data']['policy'])"

# The role is bound to mlflow-reader:
curl -s -H "X-Vault-Token: $ROOT" \
  "$VAULT_ADDR/v1/auth/approle/role/mlflow" \
  | python3 -c "import json,sys; print('token_policies:', json.load(sys.stdin)['data']['token_policies'])"

# Reproduce the wrapper's AppRole login and read with the scoped token:
ROLE_ID=$(curl -s -H "X-Vault-Token: $ROOT" \
  "$VAULT_ADDR/v1/auth/approle/role/mlflow/role-id" \
  | python3 -c "import json,sys; print(json.load(sys.stdin)['data']['role_id'])")
SECRET_ID=$(curl -s -X POST -H "X-Vault-Token: $ROOT" \
  "$VAULT_ADDR/v1/auth/approle/role/mlflow/secret-id" \
  | python3 -c "import json,sys; print(json.load(sys.stdin)['data']['secret_id'])")
SCOPED=$(curl -s -X POST \
  --data "{\"role_id\":\"$ROLE_ID\",\"secret_id\":\"$SECRET_ID\"}" \
  "$VAULT_ADDR/v1/auth/approle/login" \
  | python3 -c "import json,sys; print(json.load(sys.stdin)['auth']['client_token'])")
curl -s -H "X-Vault-Token: $SCOPED" \
  "$VAULT_ADDR/v1/secret/data/mlflow" | python3 -m json.tool
```
The first call prints the corrected HCL (`path "secret/data/mlflow"` with `read`). The second shows `token_policies: ['default', 'mlflow-reader']`. The last returns the KV entry with `admin_password` in the `data.data` block — proof the AppRole-scoped token can read the secret.

#### References

- Vault ACL policies — path rules and capabilities (`read`, `create`, `update`, …): https://developer.hashicorp.com/vault/docs/concepts/policies
- AppRole auth method — machine login with `role_id` + `secret_id`: https://developer.hashicorp.com/vault/docs/auth/approle
- KV v2 secrets engine — why data lives under the `secret/data/<name>` path: https://developer.hashicorp.com/vault/api-docs/secret/kv/kv-v2
