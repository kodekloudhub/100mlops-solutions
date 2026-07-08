# Solution

A DVC *remote* is the shared storage (S3, GCS, SSH, or — here — a SeaweedFS S3-compatible bucket) where the actual data lives. `dvc push` uploads cache objects to the remote and `dvc pull` fetches them, so teammates share data without putting it in Git. Remotes are configured in `.dvc/config`: each remote has a `url` (e.g. `s3://bucket`), connection settings such as `endpointurl` and credentials, and one remote is marked **default** (a `[core]` `remote =` entry) so a plain `dvc push`/`dvc pull` knows where to go. DVC stores objects content-addressed under `files/md5/<hash>`.

> As an MLOps engineer, you configure a DVC remote and push tracked data to shared object storage so the team works from the same artefacts — you are not judging the data; the dataset is synthetic.

#### Follow the steps below

##### 1. Observe the failure.
Move into the project and try to push so the current problem is visible.
```
cd /root/code/fraud-detection
dvc push
```
DVC reports that no default remote is configured. The first issue is the missing default. Once that is set, subsequent pushes will reveal a connection error against `:9999` and then a `NoSuchBucket` error against `dvc-wrong-bucket`.

##### 2. Inspect the existing configuration.
```
cat .dvc/config
dvc remote list
```
The configuration declares a remote called `s3` with credentials in place, but the URL points at `s3://dvc-wrong-bucket`, the endpoint URL points at the wrong port, and there is no `[core]` section nominating a default remote.

##### 3. Mark `s3` as the default remote.
```
dvc remote default s3
```

##### 4. Correct the endpoint URL.
The SeaweedFS S3 API listens on port `8333`, not `9999`.
```
dvc remote modify s3 endpointurl http://localhost:8333
```

##### 5. Correct the bucket name.
The actual bucket created by the lab platform is `dvc-storage`.
```
dvc remote modify s3 url s3://dvc-storage
```

> Every fix above can equivalently be applied by editing `.dvc/config` directly in the VS Code editor — set `[core] remote = s3`, the endpointurl line to `http://localhost:8333`, and the url line to `s3://dvc-storage`.

##### 6. Push the tracked data.
```
dvc push
```

##### 7. Verify in the SeaweedFS Filer UI.
Click the **SeaweedFS Filer** button at the top of the lab. Navigate to `/buckets/dvc-storage/` — a `files/` folder should now contain the pushed object under the `md5/<2-char-prefix>/<remaining-hash>` layout.

The same can be confirmed from the terminal:
```
AWS_ACCESS_KEY_ID=weedadmin AWS_SECRET_ACCESS_KEY=weedadmin123 \
  aws --endpoint-url=http://localhost:8333 s3 ls s3://dvc-storage/ --recursive
dvc remote list
dvc remote default
```

---

**References:**
- [DVC — `dvc remote`](https://dvc.org/doc/command-reference/remote)
- [DVC — `dvc push`](https://dvc.org/doc/command-reference/push)
