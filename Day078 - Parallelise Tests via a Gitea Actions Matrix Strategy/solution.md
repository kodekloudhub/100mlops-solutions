# Solution

A matrix strategy lets one CI job definition fan out into many parallel executions, one per variant. In this task you rewrite a serial `test` job so it runs each of the three test suites in its own parallel Gitea Actions job, then push and watch the matrix cells expand on the Checks tab.

> As an MLOps engineer, you fan the test job out across suites so they run in parallel instead of blocking each PR serially — you are not authoring the model; you are shaping how its checks execute. The suites run over synthetic data.

#### Follow the steps below

##### 1. Confirm the starting state.
From a VS Code terminal:
```
cd /root/code/fraud-detector
git log --oneline -3
ls tests/
cat .gitea/workflows/ci.yml
```
The branch is `add-test-matrix`. Three test files ship under `tests/` (`test_train.py`, `test_data_quality.py`, `test_model_contract.py`). The workflow declares a single `test` job running `pytest tests -v` serially.

##### 2. Open the pre-existing PR.
Click the **Gitea** button at the top of the lab, log in with `gitea-admin` / `gitea2026`, open the `fraud-detector` repository, and click **Pull Requests -> Convert test job to matrix strategy**. The **Checks** tab shows the first run with one `test` job containing all three suites.

##### 3. Rewrite the `test` job as a matrix.
Open `/root/code/fraud-detector/.gitea/workflows/ci.yml` in the VS Code editor. Replace the `test:` block with:
```yaml
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        suite: [train, data_quality, model_contract]
    steps:
      - uses: actions/checkout@v4
      - name: Install pytest + runtime deps
        run: pip install --break-system-packages pytest pandas numpy scikit-learn joblib
      - name: Run ${{ matrix.suite }} suite
        run: python3 -m pytest tests/test_${{ matrix.suite }}.py -v
```
Keep the `lint` job unchanged. Save.

##### 4. Commit and push.
```
git add .gitea/workflows/ci.yml
git commit -m "ci: fan test job out over matrix suites"
git push
```

##### 5. Watch the matrix expand on the Runs UI.
Return to the PR's **Checks** tab. The new run shows:
- `lint` — one job (unchanged).
- `test (train)`, `test (data_quality)`, `test (model_contract)` — three parallel jobs spawned from the matrix.

Click **Actions** in the repo header for the runs list, then click into the latest run to see all four jobs as lanes. Each matrix cell has its own log.

##### 6. Verify via Gitea's REST API.
From a VS Code terminal:
```
TOKEN=$(cat /root/.gitea/token)
SHA=$(curl -s -H "Authorization: token $TOKEN" \
  'http://localhost:3000/api/v1/repos/gitea-admin/fraud-detector/pulls?state=all&sort=newest' \
  | python3 -c "import json, sys; print(next(p['head']['sha'] for p in json.load(sys.stdin) if p['head']['ref']=='add-test-matrix'))")
curl -s -H "Authorization: token $TOKEN" \
  "http://localhost:3000/api/v1/repos/gitea-admin/fraud-detector/commits/$SHA/status" \
  | python3 -c "
import json, sys
body = json.load(sys.stdin)
print('overall:', body['state'])
for s in body.get('statuses', []):
    print(f\"  - {s['context']:40s} {s['status']}\")
"
```
The output shows `overall: success` and four per-job entries — one `lint` plus three `test (*)` cells — all `success`.

#### References

- Gitea Actions — workflow syntax (GitHub-Actions-compatible): https://docs.gitea.com/usage/actions/overview
- Matrix strategy — fanning one job out over variants (`strategy.matrix`, `${{ matrix.* }}`): https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/running-variations-of-jobs-in-a-workflow
