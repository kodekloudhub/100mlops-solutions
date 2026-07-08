# Solution

An MLflow **experiment** is the top-level container that groups related runs — typically one experiment per ML project. Logging every project into the shared `Default` experiment makes runs impossible to compare or filter, so each project gets its own named experiment. Experiments also carry **experiment-level metadata**: a free-text **description** (stored internally under the `mlflow.note.content` tag) and arbitrary **tags** (key/value pairs such as `team=ml-platform`) used to record ownership and filter the experiment list. This task creates two project experiments and labels them through the MLflow UI.

> As an MLOps engineer, you organize work into per-project experiments and label them with ownership metadata so runs stay filterable and traceable — you are not building or tuning any model here.

#### Follow the steps below

Open the MLflow UI via the **MLflow UI** button at the top of the lab.

##### 1. Switch to the Model training view.
The MLflow UI opens in the **GenAI** experience by default. The experiment list, description editor, and inline tag editor all live under the **Model training** view. Switch the `GenAI` / `Model training` toggle at the top-left of the sidebar to **Model training** before continuing.

##### 2. Confirm the starting state.
The left sidebar shows **Experiments** and **Model registry**. Click **Experiments** to open the experiments table — the seeded `legacy-models` experiment (with the `team: ml-legacy` tag visible in the Tags column) is listed alongside the platform's `Default` experiment. Both are reference material; do not edit them.

##### 3. Create the `fraud-detection` experiment.
1. In the Experiments table, click the **Create** button (top-right).
2. In the dialog, set the **Experiment Name** to `fraud-detection`.
3. Leave the artifact location unchanged unless you have a specific reason to customise it.
4. Submit the dialog with **Create**.

##### 4. Add a description to `fraud-detection`.
1. From the Experiments table, click the `fraud-detection` row to open the experiment's Overview page.
2. Open the three-dot (`⋮`) menu at the top-right of the Overview page and choose **Edit description**.
3. Type a short non-empty description — for example, `Production fraud-detection models, owned by the ML platform team.`
4. Save the description.

##### 5. Add the `team` tag to `fraud-detection`.
1. Return to the **Experiments** list via the left sidebar; the table now shows a **Tags** column.
2. Hover the `fraud-detection` row's **Tags** cell — an inline **add-tag** control (a small `+` icon) appears.
3. Click it and enter the tag as **Key** `team` and **Value** `ml-platform`, then save.

##### 6. Create the `churn-prediction` experiment.
Repeat step 3 with the name `churn-prediction`. No description is required for this experiment.

##### 7. Add the `team` tag to `churn-prediction`.
Apply the same inline-tag workflow as step 5 to the `churn-prediction` row — **Key** `team`, **Value** `analytics`.

##### 8. Verify.
Both `fraud-detection` and `churn-prediction` now appear in the Experiments table. The `fraud-detection` row shows the description text (truncated) in its Description column, and the `team` tag is visible in the Tags column. The `churn-prediction` row shows only the `team` tag.

> **Cross-check from the terminal (optional):**
> ```
> python3 -c "
> from mlflow import MlflowClient
> c = MlflowClient('http://localhost:5000')
> for name in ('fraud-detection', 'churn-prediction'):
>     e = c.get_experiment_by_name(name)
>     print(name, '→ tags:', dict(e.tags))
> "
> ```
> The output confirms the `team` tag on both experiments, and the description (stored under `mlflow.note.content`) on `fraud-detection`.

#### References
- MLflow Tracking — experiments and organising runs: https://mlflow.org/docs/latest/tracking.html
- MLflow tracking API (experiments, tags): https://mlflow.org/docs/latest/tracking/tracking-api.html
- `MlflowClient` (used in the optional terminal cross-check): https://mlflow.org/docs/latest/python_api/mlflow.client.html
