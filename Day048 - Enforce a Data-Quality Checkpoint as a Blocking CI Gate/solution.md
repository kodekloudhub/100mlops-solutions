# Solution

A data-quality check only protects the repo if it is a **gate** — a failing validation must *fail the CI job* so the PR cannot merge. A step that runs the checkpoint but is marked `continue-on-error`, or whose command swallows its exit code (`… || true`), is not a gate: the run stays green and bad data merges anyway. `src/gx_run.py` runs the `drift_check` checkpoint and **exits non-zero when the data violates the suite**, so wiring it as an ordinary step is all it takes — its failure becomes the job's failure. This task adds that gating step.

> As an MLOps engineer, you wire the data-validation checkpoint as a blocking CI merge gate so bad data can't ship—you are not analysing the dataset itself. The data is synthetic.

#### Follow the steps below

##### 1. Confirm the starting state.
From a VS Code terminal:
```
cd /root/code/fraud-detector
cat .gitea/workflows/data-quality.yml
```
The `data-quality` job checks out the repo and installs Great Expectations, but there is **no step that runs the checkpoint** — so the workflow can't catch drifted data. `src/gx_run.py` is the checkpoint runner (it exits non-zero on a failing validation).

##### 2. Open the pre-existing PR.
Click the **Gitea** button, log in with `gitea-admin` / `gitea2026`, open the `fraud-detector` repo. The **Actions** tab (top nav) shows the `data-quality` workflow installing GE but never validating the data; the pre-opened PR is under **Pull Requests → Enforce the data-quality checkpoint as a CI gate**.

##### 3. Add the gating step.
Open `/root/code/fraud-detector/.gitea/workflows/data-quality.yml` in the VS Code editor. Add a step to the end of the `data-quality` job's `steps:` list that runs the checkpoint:
```yaml
      - name: Run drift_check checkpoint (data-quality gate)
        run: python3 -m src.gx_run
```
Leave it as an ordinary step — do **not** add `continue-on-error: true` and do **not** append `|| true`. `gx_run` returns a non-zero exit code when the checkpoint fails, so an ordinary step propagates that failure to the job and blocks the merge. Save.

##### 4. Commit and push.
```
git add .gitea/workflows/data-quality.yml
git commit -m "ci: gate merges on the drift_check data-quality checkpoint"
git push
```

##### 5. Verify the gate passes on clean data.
Open the repo's **Actions** tab (top nav). A new run starts and the `data-quality` job now runs the checkpoint; the job log ends with `Checkpoint drift_check success=True` and the run goes **green** — the current `transactions.csv` is clean, so the gate lets it through. Confirm from a terminal:
```
TOKEN=$(cat /root/.gitea/token)
PR_SHA=$(curl -s -H "Authorization: token $TOKEN" \
  http://localhost:3000/api/v1/repos/gitea-admin/fraud-detector/pulls/1 \
  | python3 -c "import json,sys; print(json.load(sys.stdin)['head']['sha'])")
curl -s -H "Authorization: token $TOKEN" \
  "http://localhost:3000/api/v1/repos/gitea-admin/fraud-detector/commits/${PR_SHA}/status" \
  | python3 -c "import json,sys; print('state:', json.load(sys.stdin).get('state'))"
```
Output: `state: success`.

##### 6. Confirm the gate actually bites (the point of the lab).
A gate is only real if bad data fails it. Prove it locally against a throwaway copy — inject a negative `amount` (which violates the `amount >= 0` expectation) and run the checkpoint:
```
rm -rf /tmp/gate-demo && cp -r /root/code/fraud-detector /tmp/gate-demo && rm -rf /tmp/gate-demo/gx
python3 -c "import pandas as pd; p='/tmp/gate-demo/data/transactions.csv'; df=pd.read_csv(p); df.loc[len(df)]=[-999.0,3,1,0]; df.to_csv(p, index=False)"
cd /tmp/gate-demo && python3 -m src.gx_run; echo "exit=$?"
```
Append the bad row with pandas (not a raw `>>`, which would merge into the last line if the file has no trailing newline and trigger a CSV parse error instead of a clean validation failure). The run prints `Checkpoint drift_check success=False` and `exit=1` — the `amount >= 0` expectation rejects the `-999.0` row. In CI that non-zero exit fails the `data-quality` job, so a PR carrying that row would be blocked. That is the gate.

#### References

- Gitea Actions overview — how the runner executes `.gitea/workflows/`: https://docs.gitea.com/usage/actions/overview
- Great Expectations (GX Core) — the checkpoint runner that reports pass/fail via exit status: https://github.com/great-expectations/great_expectations
- CI gate pattern — failing a job on a quality check to block merges: https://docs.gitea.com/usage/actions/comparison
