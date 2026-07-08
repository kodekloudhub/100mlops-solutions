# Solution

A DVC pipeline is a series of *stages* declared in `dvc.yaml`. Each stage has a `cmd` (the command to run), `deps` (its inputs — the data files and scripts it reads) and `outs` (the files it produces). When one stage lists another stage's output in its `deps`, DVC chains them into a dependency graph (DAG) and runs them in the right order. `dvc repro` executes the pipeline, skips stages whose inputs haven't changed, and records exact hashes of every dep and out in `dvc.lock` so the run is reproducible. You can define stages with `dvc stage add` or by writing `dvc.yaml` by hand.

> As an MLOps engineer, you wire a reproducible data pipeline so a stage only re-runs when its inputs change — you are not judging the outputs; the processing logic is a stand-in.

#### Follow the steps below

##### 1. Inspect the scripts.
See what each script reads and writes so the stage `deps` and `outs` are correct.
```
cd /root/code/fraud-detection
cat src/data/process_data.py
cat src/data/split_data.py
```
`process_data.py` reads `data/raw/transactions.csv` and writes `data/processed/clean_transactions.csv`. `split_data.py` reads that cleaned file and writes `train.csv` and `test.csv`.

##### 2. Define the `process_data` stage.
`dvc stage add` appends a stage to `dvc.yaml` (creating the file if needed). `-n` names the stage, `-d` adds a dependency, `-o` adds an output, and the final argument is the command. Use `python3`, the binary guaranteed to be on `PATH`.
```
dvc stage add -n process_data \
  -d data/raw/transactions.csv -d src/data/process_data.py \
  -o data/processed/clean_transactions.csv \
  "python3 src/data/process_data.py"
```

##### 3. Define the `split_data` stage.
Listing the upstream output (`data/processed/clean_transactions.csv`) as a dependency is what chains the two stages, so DVC runs `process_data` first and re-runs `split_data` whenever the cleaned data changes.
```
dvc stage add -n split_data \
  -d data/processed/clean_transactions.csv -d src/data/split_data.py \
  -o data/processed/train.csv -o data/processed/test.csv \
  "python3 src/data/split_data.py"
```
> You can equivalently write these two stages by hand in `dvc.yaml` in the VS Code editor — the result is the same `stages:` block.

##### 4. Run the pipeline.
```
dvc repro
```
DVC runs `process_data` then `split_data` in dependency order, writes the outputs to `data/processed/`, and records `dvc.lock`.

##### 5. Verify.
```
cat dvc.yaml
ls -la data/processed/
dvc status
```
`data/processed/` contains `clean_transactions.csv`, `train.csv`, and `test.csv`, and `dvc status` reports "Data and pipelines are up to date."

---

**References:**
- [DVC — Defining Pipelines (`dvc.yaml`)](https://dvc.org/doc/user-guide/pipelines/defining-pipelines)
- [DVC — `dvc stage add`](https://dvc.org/doc/command-reference/stage/add)
- [DVC — `dvc repro`](https://dvc.org/doc/command-reference/repro)
