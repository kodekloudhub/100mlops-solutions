# Solution

A local ML dev stack is rarely one container — here it's three: **Jupyter** for notebooks, **MLflow** for tracking, and **SeaweedFS** as S3-compatible artifact storage, wired together with Docker **Compose**. The stack ships with two deliberate misconfigurations that leave the containers running but the UIs unreachable, and this task has you diagnose and fix them: a **`ports` mapping** entered in the wrong `host:container` order (so the SeaweedFS Filer UI isn't on its expected port), and a **missing `command:` override** (so Jupyter starts behind a generated auth token). Both are "it's up but it doesn't work" bugs — the kind Compose config drift produces in real life.

> As an MLOps engineer, you keep the local multi-service dev stack wired correctly so the team can reach every tool — you are not doing any modelling here. The stack is a development scaffold.

#### Follow the steps below

##### 1. Confirm the starting state.
From a VS Code terminal:
```
cd /root/code/ml-dev
ls
cat docker-compose.yml
```
The compose file carries `jupyter`, `mlflow`, and `seaweedfs` service blocks. The SeaweedFS ports are mapped `9001:8333` + `9000:8888` (swapped — host 9001 is bound to the S3 API container port and host 9000 is bound to the Filer UI container port), and the Jupyter service has no `command:` override.

##### 2. Run the draft stack and observe the failures.
```
docker compose up -d
docker compose ps
```
All three containers come up, but two of them are misbehaving:
```
curl -s -o /dev/null -w 'jupyter=%{http_code}\n' http://localhost:8888/
curl -s -o /dev/null -w 'mlflow=%{http_code}\n' http://localhost:5000/
curl -s -o /dev/null -w 'seaweed-filer=%{http_code}\n' http://localhost:9001/
```
`mlflow=200`. `jupyter` returns a login page — opening the **Jupyter Lab** button prompts for a token. `seaweed-filer` does not return a Filer-UI page — host port 9001 is bound to the SeaweedFS S3 API, not the Filer UI.

##### 3. Inspect `docker-compose.yml`.
Open `/root/code/ml-dev/docker-compose.yml` in the VS Code editor. The two problems:
- `seaweedfs.ports` lists `9001:8333` and `9000:8888`. SeaweedFS serves the S3 API on container port `8333` and the Filer UI on container port `8888`; the lab's convention is host `9000` for the S3 API and host `9001` for the Filer UI, so the host-to-container mapping must be `9000:8333` and `9001:8888`.
- `jupyter` has no `command:`. The image's default entrypoint starts the server with a freshly generated auth token.

##### 4. Fix the SeaweedFS port mappings.
Update the `seaweedfs` service's `ports` block so host `9000` maps to container `8333` (S3 API) and host `9001` maps to container `8888` (Filer UI):
```yaml
  seaweedfs:
    image: chrislusf/seaweedfs:4.22
    container_name: ml-seaweedfs
    ports:
      - "9000:8333"
      - "9001:8888"
    volumes:
      - seaweedfs-data:/data
    command: server -dir=/data -s3
```

##### 5. Add the Jupyter command override.
Append a `command:` line to the `jupyter` service that starts the notebook with auth disabled:
```yaml
  jupyter:
    image: jupyter/base-notebook:python-3.11
    container_name: ml-jupyter
    ports:
      - "8888:8888"
    volumes:
      - ./notebooks:/home/jovyan/work
    command: "start-notebook.sh --ServerApp.token='' --ServerApp.password=''"
```
Save the file.

##### 6. Recreate the stack and verify.
```
docker compose up -d --force-recreate
docker compose ps
```
All three containers report `Up`. Re-run the three curls from Step 2 — `jupyter=200`, `mlflow=200`, `seaweed-filer=200`.

##### 7. Open the three GUIs.
Use the buttons at the top of the lab — **Jupyter Lab** (port 8888), **MLflow UI** (5000), **SeaweedFS Filer** (9001). Jupyter opens straight into the notebook workspace with no token prompt; MLflow shows the empty experiments dashboard; the SeaweedFS Filer lists `/buckets/` (empty by default). The full dev stack is live.

#### References

- Compose file `services` reference — the `ports` (host:container) and `command` keys corrected here: https://docs.docker.com/reference/compose-file/services/
- `docker compose up` command reference: https://docs.docker.com/reference/cli/docker/compose/up/
