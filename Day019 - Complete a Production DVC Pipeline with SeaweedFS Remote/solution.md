# Solution

**About a DVC pipeline release:** A DVC pipeline is the full DAG of stages declared in `dvc.yaml` — here `ingest → validate → preprocess → train → evaluate`. `dvc repro` walks that graph, runs only the stages whose inputs changed, and records the exact hash of every dependency and output in `dvc.lock`, which is what makes the run reproducible. The large artefacts themselves (the model, processed data) live in DVC's cache rather than Git; `dvc push` uploads them to a remote — here a SeaweedFS bucket, which speaks the S3 protocol — so they are durable and shareable. Finally, a Git tag on the commit that holds `dvc.yaml`, `dvc.lock`, and the params/`.dvc` files freezes the entire code + data + model state as a named release (`v1.0`) that anyone can later check out and `dvc pull` to reproduce exactly.

> As an MLOps engineer, you complete a reproducible pipeline, push its artefacts to a remote, and freeze it as a tagged release — you are not evaluating model quality; the dataset and metrics are synthetic.

#### Follow the steps below

##### 1. Observe the current state.
In the **VS Code file explorer**, open `/root/code/ml-pipeline/` and click through `dvc.yaml`, `params.yaml`, and the `scripts-staging/` folder to see what has been pre-wired and what is still missing.

Then open the VS Code terminal and run the pipeline once so the shipped problem surfaces:
```
cd /root/code/ml-pipeline
dvc repro
```
DVC runs `ingest` and `validate` successfully, then fails at `preprocess` with:
```
ERROR: failed to reproduce 'preprocess': output 'data/processed/cleaned.csv' does not exist
```
The reason is a typo — `preprocess.outs` declares `data/processed/cleaned.csv`, but `scripts/preprocess.py` actually writes `data/processed/clean.csv`.

##### 2. Fix the preprocess output path.
Open `dvc.yaml` in the **VS Code editor**. In the `preprocess` stage, change the last line from:
```
      - data/processed/cleaned.csv
```
to:
```
      - data/processed/clean.csv
```
Save the file (`Ctrl+S`).

##### 3. Copy the remaining scripts into `scripts/`.
The `train.py` and `evaluate.py` files are pre-staged in `scripts-staging/` — visible in the file explorer. Copy them across from the terminal. Run each line on its own and wait for the prompt to return before submitting the next; pasting both lines together is fine, but do not chain them with the next step:
```
cp scripts-staging/train.py    scripts/train.py
cp scripts-staging/evaluate.py scripts/evaluate.py
```

##### 4. Declare the `train` and `evaluate` stages with `dvc stage add`.
Author the two remaining stages from the acceptance criteria. `dvc stage add` writes each stage into `dvc.yaml` for you — you translate the criteria into flags rather than hand-indenting YAML, and each command is a single line, so there is no multi-line terminal-paste hazard. Run each command on its own (wait for the prompt to return before the next).

The `train` stage — depends on the preprocessed data and its script, reads the four params, produces the model + test split, and declares `metrics.json` as a no-cache metric:
```
dvc stage add -n train \
  -d data/processed/clean.csv -d scripts/train.py \
  -p n_estimators,max_depth,test_size,random_seed \
  -o models/model.pkl -o data/processed/test_split.csv \
  -M metrics.json \
  python3 scripts/train.py
```

The `evaluate` stage — depends on the model, the test split, and its script, and writes the evaluation report as a no-cache output:
```
dvc stage add -n evaluate \
  -d models/model.pkl -d data/processed/test_split.csv -d scripts/evaluate.py \
  -O reports/evaluation.json \
  python3 scripts/evaluate.py
```

`-o` declares a **cached** output; `-M` declares a **metrics** file with `cache: false`; `-O` declares an output with `cache: false`. Open `dvc.yaml` afterwards and confirm it now ends with a `train:` stage (with `params`, `outs`, and a `metrics.json` entry under `metrics:` with `cache: false`) followed by an `evaluate:` stage (with `reports/evaluation.json` under `outs:` with `cache: false`).

##### 5. Run the full pipeline.
```
dvc repro
```
DVC executes every stage in order, regenerates `dvc.lock`, and produces the model, metrics, and report files. You should see five stages reach completion.

##### 6. Push artefacts to the SeaweedFS remote.
The remote is pre-configured in `.dvc/config`. Push the cache so the bucket holds every tracked artefact:
```
dvc push
```

##### 7. Commit every change and tag the release as `v1.0`.
```
git add .
git commit -m "Complete production DVC pipeline"
git tag v1.0
```

##### 8. Verify.
Confirm the pipeline is clean, the metrics surface, the bucket is populated, and the tag exists:
```
dvc status
dvc metrics show
git tag -l
AWS_ACCESS_KEY_ID=weedadmin AWS_SECRET_ACCESS_KEY=weedadmin123 \
  aws --endpoint-url=http://localhost:8333 s3 ls s3://dvc-storage/ --recursive | head
```
Then open the **SeaweedFS Filer** button at the top of the lab, navigate to `/buckets/dvc-storage/`, and confirm it now holds the pushed artefacts under the `files/md5/...` layout.

---

**References:**
- [DVC — Defining Pipelines (`dvc.yaml` stages)](https://dvc.org/doc/user-guide/pipelines/defining-pipelines)
- [DVC — `dvc repro`](https://dvc.org/doc/command-reference/repro)
- [DVC — `dvc push`](https://dvc.org/doc/command-reference/push)
- [DVC — Amazon S3 / S3-compatible remotes (`endpointurl`)](https://dvc.org/doc/user-guide/data-management/remote-storage/amazon-s3)
