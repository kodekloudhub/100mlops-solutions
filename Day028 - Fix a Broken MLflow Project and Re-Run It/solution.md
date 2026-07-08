# Solution

An **MLflow Project** packages training code so it runs reproducibly with one `mlflow run` command. The `MLproject` file is the descriptor: each **entry point** declares typed **parameters** (with defaults) and a **`command:`** that substitutes those parameters — written as `{name}` placeholders — into a shell invocation of the script. MLflow fills the placeholders from the parameter defaults or from `-P name=value` overrides, then runs the command. The contract is exact: every flag in the `command:` must match the flag names the script's argparse declares, or the run fails before training starts. Here the shipped `command:` forwards only one parameter, under a wrong flag name — so the job is to author the full command against `train.py`'s interface, then run it.

> As an MLOps engineer, you fix the MLproject packaging so a training run reproduces from a single `mlflow run` command with every parameter wired correctly — you are not improving the model; the trainer's outputs are synthetic.

#### Follow the steps below

##### 1. Observe the failure in the MLflow UI.
Open the **MLflow UI** button at the top of the lab. Under **Model training → Experiments → trainer**, a single **FAILED** run is already present. The full stderr is also captured at `/tmp/mlflow-run-initial.log` on the controlplane.

Reproduce the error from the terminal:
```
cd /root/code/trainer
export MLFLOW_TRACKING_URI=http://localhost:5000
mlflow run . -e train --env-manager=local
```
The output includes:
```
train.py: error: unrecognized arguments: --n_est 100
```

##### 2. Inspect the MLproject command and train.py's interface.
Open `/root/code/trainer/MLproject` in the VS Code editor. The `train` entry point declares four parameters (`n_estimators`, `max_depth`, `test_size`, `random_seed`), but its `command:` forwards only one of them — under the wrong flag name:
```
command: >
  python train.py
  --n_est {n_estimators}
```
Now open `train.py`. Its argparse (run with `allow_abbrev=False`, so prefixes like `--n_est` are **not** silently accepted) declares the authoritative flags:
```
--n_estimators   --max_depth   --test_size   --random_seed
```
Two problems: the one flag present (`--n_est`) does not match argparse (hence the `unrecognized arguments: --n_est 100` failure), and the other three parameters are never passed at all.

##### 3. Author the complete MLproject command.
`train.py` is authoritative — do not modify it. Write the `command:` so it forwards **all four** declared parameters, each under the flag name argparse expects:
```
command: >
  python train.py
  --n_estimators {n_estimators}
  --max_depth {max_depth}
  --test_size {test_size}
  --random_seed {random_seed}
```
Save the file. Each `{name}` is substituted by MLflow from the parameter default or the matching `-P` override at run time.

##### 4. Run the project with explicit parameters.
From the project directory, use `mlflow run` with `-P` to override the defaults. Point the environment at the local tracking server and the `trainer` experiment:
```
cd /root/code/trainer
export MLFLOW_TRACKING_URI=http://localhost:5000
MLFLOW_EXPERIMENT_NAME=trainer mlflow run . -e train \
  -P n_estimators=200 -P max_depth=10 --env-manager=local
```
The run completes successfully. The MLflow UI now shows a new **FINISHED** run in the `trainer` experiment whose `params.n_estimators` is `200`.

##### 5. Run the project with default parameters.
Run the same command again, this time with no `-P` overrides — the defaults (`n_estimators=100`, `max_depth=5`) apply:
```
MLFLOW_EXPERIMENT_NAME=trainer mlflow run . -e train --env-manager=local
```

##### 6. Verify.
The `trainer` experiment now contains:
- the original **FAILED** run from the lab startup;
- one **FINISHED** run with `params.n_estimators=200`;
- one **FINISHED** run with `params.n_estimators=100`.

Open the MLflow UI run list to confirm all three. Both successful runs carry logged `accuracy` and `f1_score` metrics and a `model/` artefact. The explicit run shows `params.max_depth=10`, confirming the `--max_depth {max_depth}` you authored forwards the `-P max_depth=10` override through to `train.py`.

#### References
- MLflow Projects — the `MLproject` file, entry points, and parameters: https://mlflow.org/docs/latest/projects.html
- `mlflow run` CLI (`-e`, `-P`, `--env-manager`): https://mlflow.org/docs/latest/cli.html
