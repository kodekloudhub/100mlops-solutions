# Solution

Shipping a model image by hand is fine once; a pipeline does it the same way every time. This capstone fixes a shell-based Docker CI pipeline (`build.sh`) that chains four stages under `set -euo pipefail`: run the tests, build the image, tag it with the **short git SHA** (so every build is traceable to a commit), and push it to the private registry. Three integration bugs stop it — a test-discovery path that doesn't exist, an undefined shell variable that aborts the run under `set -u`, and a registry endpoint pointing at the wrong port. You surface them one at a time by re-running the script until the SHA-tagged image lands in the registry. Git-SHA tagging is the idea that carries forward: `:latest` is ambiguous, but `:<sha>` pins exactly which code produced the image.

> As an MLOps engineer, you automate build-tag-push so releases are repeatable and every image traces back to a commit — you are not evaluating the model. The image content is a synthetic stand-in.

#### Follow the steps below

##### 1. Confirm the starting state.
From a VS Code terminal:
```
cd /root/code/ci
docker ps --filter name=local-registry
ls app/
cat build.sh
```
`local-registry` is running on port 5555. The `app/` directory carries `app.py`, `test_app.py`, `Dockerfile`, and a git repo. The `build.sh` script has four numbered stages (test / build / tag / push).

**Every fix in this task is in the one file `/root/code/ci/build.sh`.** The `app/` sources, tests, and Dockerfile are all correct — you do not edit them. There are three separate problems in `build.sh`; the steps below surface and fix them one at a time by re-running the script.

##### 2. Run the pipeline and surface the first issue.
```
./build.sh
```
Stage 1 aborts:
```
[ci] stage 1/4 — running tests
ERROR: file or directory not found: app/tests/
```
The tests live at `app/test_app.py`, not under an `app/tests/` directory.

##### 3. Fix the pytest path.
Open `/root/code/ci/build.sh` in the VS Code editor. Change Stage 1's pytest argument:
```bash
python3 -m pytest app/
```
Save the file. pytest will auto-discover `test_*.py` under `app/`.

##### 4. Re-run the pipeline and surface the second issue.
```
./build.sh
```
Stage 1 passes three tests. Stage 2 builds the image. Stage 3 aborts:
```
[ci] stage 3/4 — tagging
./build.sh: line 23: GIT_SHA: unbound variable
```
The script computes `SHA=$(git -C app rev-parse --short HEAD)` but tags with `$GIT_SHA`. Under `set -u`, the undefined variable halts execution immediately.

##### 5. Fix the SHA variable reference.
Still in `/root/code/ci/build.sh`, change the tag line to reference the variable the script actually defines:
```bash
TAGGED="$REGISTRY/$IMAGE:$SHA"
```
Save the file.

##### 6. Re-run the pipeline and surface the third issue.
```
./build.sh
```
Stage 3 now tags cleanly. Stage 4 aborts:
```
[ci] stage 4/4 — pushing
Error response from daemon: Get "http://localhost:5000/v2/": dial tcp 127.0.0.1:5000: connect: connection refused
```
Nothing is listening on port 5000 — the lab's registry is on 5555.

##### 7. Fix the registry port.
Still in `/root/code/ci/build.sh`, update the `REGISTRY` variable:
```bash
REGISTRY="localhost:5555"
```
Save the file.

##### 8. Re-run the pipeline and verify.
```
./build.sh
```
All four stages complete. The final line prints `[ci] complete: localhost:5555/ml-ci-app:<sha>`. Confirm the image reached the registry:
```
curl -s http://localhost:5555/v2/_catalog
curl -s http://localhost:5555/v2/ml-ci-app/tags/list
```
The first returns `{"repositories":["ml-ci-app"]}`. The second returns the list of tags — including the current short git SHA for `app/`. The CI pipeline is green end-to-end.

#### References

- `git rev-parse --short HEAD` — deriving the short commit SHA used as the image tag: https://git-scm.com/docs/git-rev-parse
- `docker push` — publishing the SHA-tagged image to the registry: https://docs.docker.com/reference/cli/docker/image/push/
- Registry HTTP API v2 — the `/v2/_catalog` and `/v2/<name>/tags/list` endpoints the checks query: https://distribution.github.io/distribution/spec/api/
