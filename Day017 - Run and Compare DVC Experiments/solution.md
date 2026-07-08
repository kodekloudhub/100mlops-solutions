# Solution

A DVC experiment is a single run of the pipeline with one or more parameter values changed. `dvc exp run --set-param <key>=<value>` overrides a value from `params.yaml` for that run only — it does not edit the file — then re-executes the affected stages and records the run's parameters and metrics under a generated experiment name. Because every experiment is tracked, you can list them side by side with `dvc exp show`, compare their metrics, and promote the best one back into your working tree with `dvc exp apply <name>`, which updates `params.yaml`, `metrics.json`, and the model artefact to match that run. Here you vary `max_depth` (the maximum depth of each tree in the random forest): too shallow underfits the data, too deep overfits it, so the held-out `f1_score` rises and then falls — giving a genuine "best" experiment to find.

> As an MLOps engineer, you run tracked experiments and promote the chosen one into version control so the run is reproducible — you are not chasing model quality; the dataset and metrics are synthetic.

#### Follow the steps below

##### 1. Move into the project.
```
cd /root/code/fraud-detection
```
The baseline pipeline has already been run once with `max_depth: 4`. Check its score so you have something to compare against:
```
dvc metrics show
```

##### 2. Run three experiments with different `max_depth` values.
Each command overrides `max_depth` for a single run, retrains the model, and records the result as a tracked experiment. Use a range that spans shallow to deep so the under/overfitting trade-off is visible.
```
dvc exp run --set-param max_depth=2
dvc exp run --set-param max_depth=6
dvc exp run --set-param max_depth=12
```

##### 3. Compare the experiments.
List every experiment side by side with its parameters and metrics.
```
dvc exp show
```
Each row shows its `max_depth` and the resulting `f1_score`. Identify the experiment with the highest `f1_score` and note its name (the generated label in the leftmost column, for example `gabby-caps`).

##### 4. Apply the best experiment.
Promote the chosen experiment's `max_depth`, `metrics.json`, and trained model into the working tree so they become the tracked state.
```
dvc exp apply <experiment-name>
```
Replace `<experiment-name>` with the name of the highest-scoring run from the previous step.

##### 5. Verify.
Confirm the workspace now reflects the applied experiment.
```
cat params.yaml
cat metrics.json
dvc status
```
`params.yaml` should hold the `max_depth` of the applied experiment (no longer `4`), `metrics.json` should report that experiment's accuracy and f1 values, and `dvc status` should report nothing to do.

> The DVC extension's **EXPERIMENTS** view (open the DVC panel from the Activity Bar) lists every experiment alongside its parameters and metrics, which is a convenient way to compare runs at a glance. Run and apply experiments with the `dvc exp` commands shown here.

---

**References:**
- [DVC — `dvc exp run`](https://dvc.org/doc/command-reference/exp/run) (documents `--set-param`)
- [DVC — `dvc exp show`](https://dvc.org/doc/command-reference/exp/show)
- [DVC — `dvc exp apply`](https://dvc.org/doc/command-reference/exp/apply)
- [DVC — Running Experiments](https://dvc.org/doc/user-guide/experiment-management/running-experiments)
