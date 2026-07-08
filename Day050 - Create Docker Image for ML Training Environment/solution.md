# Solution

A model that trains on one engineer's laptop is worthless if a teammate can't reproduce the run. A **Docker image** fixes the environment — the Python version, the libraries, the code — so the training job runs identically on any host. This task authors a `Dockerfile` from scratch to package an ML training environment: a base image, a working directory, the dependency install, the code copy, and the default command. The two ML-specific decisions are the base-image choice (`slim`, not `alpine`, because scikit-learn ships no musl wheel) and installing the full import closure — including the runtime-only `joblib` the trainer uses to persist the model.

> As an MLOps engineer, you package the training environment into a reproducible image so any host reproduces the run identically — you are not tuning the model itself. The accuracy and F1 the container prints are computed on a small synthetic dataset.

#### Follow the steps below

##### 1. Confirm the starting state.
From a VS Code terminal:
```
cd /root/code/ml-docker
ls
cat Dockerfile
```
`train.py` is complete. `Dockerfile` is a scaffold of five numbered TODOs (base image, working directory, dependency install, copy, command) with no runnable instructions yet — `docker build` would fail until they are authored.

##### 2. Author the Dockerfile.
Open `/root/code/ml-docker/Dockerfile` in the VS Code editor and write the five instructions to the team standard:
```dockerfile
FROM python:3.11-slim

WORKDIR /app

RUN pip install --no-cache-dir scikit-learn pandas numpy joblib

COPY train.py /app/train.py

CMD ["python3", "train.py"]
```
Two details matter:
- **`python:3.11-slim`, not `python:3.11-alpine`.** Alpine uses musl libc, for which scikit-learn ships no manylinux wheel — `pip install` then fails outright or falls back to a multi-minute source build that exceeds the lab's memory budget. The `slim` (Debian) base loads the prebuilt wheels directly.
- **`joblib` on the install line.** `train.py` persists the fitted model with `joblib.dump(...)`. Omitting it builds a working image that then aborts at runtime with `ModuleNotFoundError: No module named 'joblib'`.

Save the file.

##### 3. Build the image.
```
docker build -t ml-trainer:v1 .
```
The build runs each instruction in turn and completes once all four wheels install.

##### 4. Run the container.
```
docker run --rm ml-trainer:v1
```
The container prints the accuracy, the F1 score, and `Model saved to /app/model.pkl`.

##### 5. Verify the image and its imports.
```
docker images ml-trainer:v1
docker run --rm ml-trainer:v1 python3 -c "import sklearn, pandas, numpy, joblib; print('OK')"
```
`docker images` lists the tagged image. The second command prints `OK` — every package the training code imports resolves inside the image.

#### References

- Dockerfile reference — the `FROM` / `WORKDIR` / `RUN` / `COPY` / `CMD` instructions authored here: https://docs.docker.com/reference/dockerfile/
- The official `python` image and its `-slim` variants: https://hub.docker.com/_/python
- `docker build` command reference: https://docs.docker.com/reference/cli/docker/buildx/build/
