# Solution

An image that only exists on the machine that built it can't be deployed anywhere. A **container registry** is the distribution point — downstream clusters pull images from it by tag. This task publishes the `fraud-detector` image to a private registry (`registry:2`, running on host port `5555`) by authoring the publish flow in `push.sh`: name the built image for the registry (`host:port/name:tag`) and `docker push` it, then confirm it landed via the registry's HTTP API (`/v2/_catalog` and `/v2/<name>/tags/list`). The detail that trips people up is that `docker tag` only writes local metadata — nothing reaches the registry until the push runs.

> As an MLOps engineer, you publish the model image to a registry so it's pullable across the platform — you are not changing the model. The image content is a stand-in for a real trained artifact.

#### Follow the steps below

##### 1. Confirm the starting state.
From a VS Code terminal:
```
cd /root/code/ml-registry
docker ps --filter name=local-registry
curl -s http://localhost:5555/v2/_catalog
cat push.sh
```
`local-registry` is running on host port `5555`. The catalogue responds with `{"repositories":[]}` — the registry is empty. `push.sh` builds the image but leaves the registry publish flow as a `# TODO`.

##### 2. Run the scaffold and observe the empty catalogue.
```
./push.sh
```
The script builds `fraud-detector:v1` but does nothing else — no tag for the registry, no push. Re-query the registry:
```
curl -s http://localhost:5555/v2/_catalog
```
Still `{"repositories":[]}`.

##### 3. Author the publish flow.
Open `/root/code/ml-registry/push.sh` in the VS Code editor. Replace the `# TODO` block with the three-step publish flow — set the registry host:port, tag the image for it, and push:
```bash
REGISTRY="localhost:5555"

docker tag "$IMAGE" "$REGISTRY/$IMAGE"
docker push "$REGISTRY/$IMAGE"

echo "Pushed $IMAGE to $REGISTRY"
```
Two details matter:
- **`localhost:5555`, not `localhost:5000`.** 5000 is Docker's conventional example port; this task's `registry:2` is bound to host port `5555`.
- **`docker push` is required.** `docker tag` only writes local metadata — nothing reaches the registry until the push runs.

Save the file.

##### 4. Re-run the script.
```
./push.sh
```
The build is cached; the image is tagged for `localhost:5555` and `docker push` sends every layer to the registry. The final log line confirms the push completed.

##### 5. Verify the registry catalogue.
```
curl -s http://localhost:5555/v2/_catalog
curl -s http://localhost:5555/v2/fraud-detector/tags/list
```
The first returns `{"repositories":["fraud-detector"]}`; the second returns `{"name":"fraud-detector","tags":["v1"]}`. The image is live in the private registry and pull-ready for downstream clusters.

#### References

- `docker push` — publishing a tagged image to a registry: https://docs.docker.com/reference/cli/docker/image/push/
- `docker tag` — naming an image for a target registry (`host:port/name:tag`): https://docs.docker.com/reference/cli/docker/image/tag/
- Registry HTTP API v2 — the `/v2/_catalog` and `/v2/<name>/tags/list` endpoints: https://distribution.github.io/distribution/spec/api/
