# Solution

A safe rollback undoes a bad change through the same reviewed, CI-checked path it arrived on — no force-pushes, no command-line panic reverts. In this task a pre-merged PR left `main` red, so you use Gitea's Revert button to open a revert PR, let CI verify the reverted state, and merge it to bring `main` back to green.

> As an MLOps engineer, you roll a bad merge off `main` through an auditable revert PR that re-runs CI before it lands — you are not editing the model; you are running the rollback playbook. The failing change is in synthetic training code.

#### Follow the steps below

##### 1. Confirm the broken starting state (GUI).
Click the **Gitea** button, log in with `gitea-admin` / `gitea2026`, open the `fraud-detector` repository:

- **Actions** tab → the latest run on `main` has a red ❌ — click it to see the failing lint step.
- **Pull Requests** tab → switch to **Closed** → `Add speculative hashing scaffold` is listed with a purple *Merged* pill; that's the commit that broke main.

##### 2. Confirm the same state from the terminal (optional cross-check).
For readers who want the API view:
```
TOKEN=$(cat /root/.gitea/token)
curl -s -H "Authorization: token $TOKEN" \
  'http://localhost:3000/api/v1/repos/gitea-admin/fraud-detector/pulls?state=closed&sort=newest' \
  | python3 -c "
import json, sys
for p in json.load(sys.stdin):
    print(f\"#{p['number']:>3} {p['title'][:50]:50s} merged={p.get('merged')}\")
"

SHA=$(curl -s -H "Authorization: token $TOKEN" \
  'http://localhost:3000/api/v1/repos/gitea-admin/fraud-detector/branches/main' \
  | python3 -c "import json, sys; print(json.load(sys.stdin)['commit']['id'])")

curl -s -H "Authorization: token $TOKEN" \
  "http://localhost:3000/api/v1/repos/gitea-admin/fraud-detector/commits/$SHA/status" \
  | python3 -c "import json, sys; print('main HEAD CI state:', json.load(sys.stdin).get('state'))"
```
The `Add speculative hashing scaffold` PR is `merged=True`. Main's HEAD CI state is `failure` (or `pending` if the runner is still catching up).

##### 3. Open the merged PR and jump to its commit.
Click **Pull Requests** → **Closed** tab → open `Add speculative hashing scaffold` (`#1`). On the **Conversation** tab, find the commit that landed on main (shown as `gitea-admin added 1 commit …` with a short SHA on the right, e.g. `7714af0508`). Click that short SHA — it opens the commit-detail page.

##### 4. Open the commit's **Revert** dialog.
Gitea 1.22 exposes revert at the commit level, not the PR page. On the commit-detail page, click the blue **Operations ▾** dropdown (top-right of the commit header). The dropdown lists:
- Create branch
- Create tag
- **Revert** ← click this
- Cherry-pick

A small popup opens labelled **Revert: `<sha>`** with a **Select branch to revert onto** dropdown. Pick **`main`** (the branch that currently carries the regression).

##### 5. Fill in the Commit Changes form correctly.
Gitea opens a full-page **Commit Changes** form. Two things need attention:

1. **Edit the commit title** (first text field). By default Gitea fills it with something like `revert <sha>`. Change it to a title that starts with `Revert`, e.g.:
   ```
   Revert "feat(train): speculative hashing scaffold"
   ```
   Why: the task's rollback policy requires the eventual PR title to start with `Revert`, and Gitea derives the PR title from this commit subject.

2. **Pick the second radio**: **`Create a new branch for this commit and start a pull request`**. Do **not** leave the default *"Commit directly to the main branch"* selected — that would bypass review, which is exactly what the task's rollback policy forbids.

Click the blue button at the bottom — **Propose file change** (Gitea renames *Commit Changes* → *Propose file change* when the new-branch radio is selected). Gitea creates a branch (defaults to `gitea-admin-patch-<N>`; rename it if you want), applies the revert commit, and lands you on a **compare** page.

##### 6. Open the pull request.
The compare page shows:
- **merge into:** `gitea-admin:main`
- **pull from:** `gitea-admin:gitea-admin-patch-<N>`
- The revert commit with its diff preview.

Click the blue **New Pull Request** button. Gitea opens the PR title/body form — the title is prefilled from the commit subject (`Revert "feat(train): speculative hashing scaffold"`); edit it here if it does not already start with `Revert`. Click **Create Pull Request** at the bottom to open the PR.

##### 7. Merge the revert PR.
The new PR page opens with a diff that is the exact inverse of the original PR's diff — the unused import is gone, the long trailing comment is gone, and the `lint` + `test` checks pass within a few seconds. Once both checks are green, click **Create merge commit** → confirm by clicking **Create merge commit** again.

##### 8. Watch CI go green on main.
Return to the repo's **Actions** tab. A new `CI` run kicks off on `main` for the revert-merge commit. It finishes green — both `lint` and `test` jobs pass.

##### 9. Verify via Gitea's REST API.
From a VS Code terminal:
```
TOKEN=$(cat /root/.gitea/token)

curl -s -H "Authorization: token $TOKEN" \
  'http://localhost:3000/api/v1/repos/gitea-admin/fraud-detector/pulls?state=closed&sort=newest' \
  | python3 -c "
import json, sys
for p in json.load(sys.stdin):
    print(f\"#{p['number']:>3} {p['title'][:50]:50s} merged={p.get('merged')}\")
"

curl -s -H "Authorization: token $TOKEN" \
  'http://localhost:3000/api/v1/repos/gitea-admin/fraud-detector/commits?sha=main&limit=3' \
  | python3 -c "
import json, sys
for c in json.load(sys.stdin):
    print(c['sha'][:8], c['commit']['message'].split(chr(10))[0])
"

SHA=$(curl -s -H "Authorization: token $TOKEN" \
  'http://localhost:3000/api/v1/repos/gitea-admin/fraud-detector/branches/main' \
  | python3 -c "import json, sys; print(json.load(sys.stdin)['commit']['id'])")
curl -s -H "Authorization: token $TOKEN" \
  "http://localhost:3000/api/v1/repos/gitea-admin/fraud-detector/commits/$SHA/status" \
  | python3 -c "import json, sys; print('main HEAD CI state:', json.load(sys.stdin).get('state'))"
```
The first call lists two merged PRs — the original and the `Revert ...` one. The second shows `main`'s HEAD commit message starts with `Revert`. The third prints `main HEAD CI state: success`.

#### References

- Gitea — pull requests (opening, reviewing, merging): https://docs.gitea.com/usage/pull-request
- Reverting a pull request — the audit-traceable rollback workflow (concept, mirrored by Gitea's Revert): https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/incorporating-changes-from-a-pull-request/reverting-a-pull-request
