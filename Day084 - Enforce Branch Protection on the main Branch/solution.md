# Solution

This capstone hardens `fraud-detector` for production governance in two moves. First you finish the CI pipeline so it runs the test suite as a `test` check, not just `lint` — branch protection can only *require* a status check that actually exists, so the check has to be built and run before you can enforce it. Then you configure Gitea branch protection on `main` so direct pushes are blocked and every change must land through a pull request carrying the green `lint` and `test` checks plus at least one approving review.

> As an MLOps engineer, you build the checks that guard `main` and then lock the branch behind them, so nothing reaches production-tracking `main` unverified — you are not touching the model; you are enforcing the governance the pipeline assumes. The checks run over synthetic training code.

#### Follow the steps below

##### 1. Confirm the starting state.
From a VS Code terminal:
```
cd /root/code/fraud-detector
cat .gitea/workflows/ci.yml
TOKEN=$(cat /root/.gitea/token)
curl -s -H "Authorization: token $TOKEN" \
  http://localhost:3000/api/v1/repos/gitea-admin/fraud-detector/branch_protections \
  | python3 -m json.tool
```
`ci.yml` defines only a `lint` job — the `test` job is a `# TODO`. The `branch_protections` array is empty. So today nothing runs the tests, and nothing stops a direct push to `main`.

##### 2. Add the `test` job to the CI workflow.
Open `/root/code/fraud-detector/.gitea/workflows/ci.yml` and replace the `# TODO` block with a real `test` job that runs the repo's pytest suite:
```yaml
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install pytest + runtime deps
        run: pip install --break-system-packages pytest pandas numpy scikit-learn joblib
      - name: Run all tests
        run: python3 -m pytest tests -v
```
Save the file.

##### 3. Push it and confirm both checks run.
`main` is not protected yet, so push the pipeline fix directly:
```
git add .gitea/workflows/ci.yml
git commit -m "ci: run the test suite alongside lint"
git push origin main
```
Open the **Gitea** button, log in with `gitea-admin` / `gitea2026`, open `fraud-detector`, and click the **Actions** tab. The new run shows two green jobs — `lint` and `test`. That run is what registers `test` as a status-check context Gitea can require in the next step.

##### 4. Open the branch-protection form.
- In the repo, click **Settings** (top right).
- Left navigation: click **Branches**.
- Click **Add Rule** (or **Edit** if a placeholder rule already exists).

##### 5. Configure the rule.
Walk the form top to bottom:

**Patterns**
- **Protected Branch Name Pattern:** `main`.
- Leave **Protected file patterns** and **Unprotected file patterns** empty.

**Push**
- Pick the **Whitelist Restricted Push** radio (default is *Enable Push* — change it).
- Leave the **Whitelisted users for pushing** field empty. An empty whitelist is the rule: no one can push directly, every change must arrive through a pull request.
- Leave **Whitelist deploy keys with write access to push** unchecked.
- Leave **Require Signed Commits** unchecked.

**Pull Request Approvals**
- **Required approvals:** change `0` to **`1`**.
- Leave **Restrict approvals to whitelisted users or teams**, **Dismiss stale approvals**, and **Ignore stale approvals** at defaults (off).

**Status checks**
- Tick **Enable Status Check**.
- In the **Status check patterns** multi-line field, add two patterns (one per line):
  ```
  *lint*
  *test*
  ```
  The "Status checks found in the last week for this repository" table below the field shows what Gitea has seen — `CI / lint (push)` and `CI / test (push)`. The globs match both regardless of the trigger (`push` or `pull_request`).

**Pull Request Merge**
- Leave **Enable Merge** selected.
- Leave **Block merge on rejected reviews**, **Block merge on official review requests**, and **Block merge if pull request is outdated** at defaults (off) — you can tighten later.

Click **Save Rule** at the bottom of the form.

##### 6. Verify via the UI.
The Branches page now shows `main` with a padlock icon. Hovering reveals "Protected". Clicking the rule edit button re-opens the form with the saved values.

##### 7. Verify via Gitea's REST API.
From a VS Code terminal:
```
TOKEN=$(cat /root/.gitea/token)

curl -s -H "Authorization: token $TOKEN" \
  http://localhost:3000/api/v1/repos/gitea-admin/fraud-detector/branch_protections \
  | python3 -c "
import json, sys
for p in json.load(sys.stdin):
    print('branch:           ', p.get('branch_name'))
    print('enable_status:    ', p.get('enable_status_check'))
    print('status_contexts:  ', p.get('status_check_contexts'))
    print('required_approvals:', p.get('required_approvals'))
    print('enable_push:      ', p.get('enable_push'))
    print('enable_push_wl:   ', p.get('enable_push_whitelist'))
    print('push_wl_users:    ', p.get('push_whitelist_usernames'))
"
```
The output shows `main` protected, status checks enabled with `lint` + `test` in the contexts list, required approvals `>=1`, and the push whitelist enabled with an empty user list.

##### 8. (Optional) Exercise the guardrail.
Try pushing a commit directly to `main` from the clone at `/root/code/fraud-detector`:
```
cd /root/code/fraud-detector
git commit --allow-empty -m "probe: direct push" && git push origin main
```
Gitea refuses the push with `remote: error: ... protected branch ...` and `! [remote rejected]`. The only way onto `main` from here on is through a PR with green `lint` + `test` checks and one approving review.

#### References

- Gitea — protected branches (push whitelist, required status checks, required approvals): https://docs.gitea.com/usage/access-control/protected-branches
- Gitea Actions — workflow syntax (jobs, steps) for the `test` job: https://docs.gitea.com/usage/actions/overview
- About protected branches — the governance concept (required checks, required reviews), mirrored by Gitea: https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches
