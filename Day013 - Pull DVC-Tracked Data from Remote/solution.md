# Solution

When you clone a repo, Git brings the code and the small `.dvc` *pointer* files, but **not** the data itself — that lives in the DVC remote. `dvc pull` reads each pointer and downloads the matching object from the remote into your local cache and working tree. To reach an S3-compatible remote, DVC's client needs **credentials** (`access_key_id` / `secret_access_key`); without them the pull cannot authenticate. Credentials are set per-remote with `dvc remote modify`, separate from the connection details (URL/endpoint) configured earlier. *Best practice:* for real secrets use `dvc remote modify --local …`, which writes to the gitignored `.dvc/config.local` so credentials are never committed to Git — this task uses the plain `.dvc/config` for simplicity.

> As an MLOps engineer, you restore a dataset on a fresh clone by fixing the remote's credentials so `dvc pull` can authenticate — you are not analysing the data; the dataset is synthetic.

#### Follow the steps below

##### 1. Observe the failure.
Move into the project and try to pull the dataset.
```
cd /root/code/fraud-detection
dvc pull
```
DVC reports an authentication error against SeaweedFS — the underlying S3 client cannot find any credentials. The remote URL, endpoint, and default-remote declaration are all in place; only the credentials are missing.

##### 2. Inspect the configuration.
```
cat .dvc/config
```
The `['remote "s3"']` block lists `url` and `endpointurl` but no `access_key_id` or `secret_access_key`.

##### 3. Add the credentials.
The team's SeaweedFS credentials are `weedadmin` / `weedadmin123`. They can be added through `dvc remote modify` from the CLI, or by editing `.dvc/config` directly in the VS Code editor.
```
dvc remote modify s3 access_key_id weedadmin
dvc remote modify s3 secret_access_key weedadmin123
```

##### 4. Pull the dataset.
```
dvc pull
```
DVC fetches the dataset from SeaweedFS and restores it under `data/raw/transactions.csv`. The local DVC cache is rebuilt at the same time.

##### 5. Verify.
Confirm the dataset is back on disk, its content matches the `.dvc` pointer, and the local cache is populated.
```
ls -la data/raw/transactions.csv
dvc status data/raw/transactions.csv.dvc
find .dvc/cache -type f | head
```
The same can be confirmed in the **SeaweedFS Filer** UI: navigate to `/buckets/dvc-storage/` and the object is still present — the bucket is unchanged because pull only reads from the remote.

---

**References:**
- [DVC — `dvc pull`](https://dvc.org/doc/command-reference/pull)
- [DVC — Amazon S3 remotes (`access_key_id`, `secret_access_key`, `endpointurl`)](https://dvc.org/doc/user-guide/data-management/remote-storage/amazon-s3)
