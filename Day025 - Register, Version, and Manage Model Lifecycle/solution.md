# Solution

The MLflow **Model Registry** is a central catalogue that sits on top of run artifacts. Registering a logged model under a name (`fraud-detector`) creates a **registered model**; each subsequent registration of that name becomes a new sequential **version** (v1, v2, …), each pointing back at its source run. Rather than hard-coding "which version is in production," teams attach **aliases** — movable labels such as `challenger` and `champion` — to specific versions; serving code then loads `models:/fraud-detector@champion` and the ops team promotes a new model simply by repointing the alias. This task registers two runs as two versions and labels them with those lifecycle aliases.

> As an MLOps engineer, you promote runs into the registry as versions and move lifecycle aliases so downstream consumers pick up a new model without redeploying — you are not evaluating model quality; the runs' metrics are synthetic.

#### Follow the steps below

Open the MLflow UI via the **MLflow UI** button at the top of the lab, and switch the `GenAI` / `Model training` toggle at the top-left of the sidebar to **Model training** before continuing.

##### 1. Confirm the starting state.
From the Experiments sidebar, open `fraud-detection`. Two runs (`baseline`, `improved`) are listed in the **Runs** view. Switch to the experiment's **Models** tab (left-side nav under the experiment) — two logged-model entries, both named `model`, appear: one sourced from `baseline` and one from `improved`. Clicking into either one shows a flat artefact list (`MLmodel`, `model.pkl`, `conda.yaml`, `python_env.yaml`, `requirements.txt`) with no wrapping `model/` directory. That flat layout is MLflow 3.x's "logged model" shape.

##### 2. Register the `baseline` logged model as version 1 of `fraud-detector`.
1. From the `fraud-detection` experiment's **Models** tab, click the row whose **Source run** is `baseline`.
2. On that model's page, click **Register model** (top-right).
3. In the dialog, choose **Create New Model** and set the model name to `fraud-detector`.
4. Click **Register**. The baseline logged model becomes version 1 of the new registered model.

##### 3. Register the `improved` logged model as version 2 of `fraud-detector`.
1. Go back to the experiment's **Models** tab and click the row whose **Source run** is `improved`.
2. Click **Register model** (top-right).
3. In the dialog, choose **Select Existing Model** and pick `fraud-detector` from the dropdown.
4. Click **Register**. The improved logged model becomes version 2 of the same registered model.

##### 4. Open the registered model from the Model registry.
In the left sidebar, open **Model registry** and select `fraud-detector`. The page lists both versions.

##### 5. Add the model-level description.
1. On the `fraud-detector` registry page, click the **Description** field (or its edit pencil).
2. Type a short description referencing the word `fraud` — for example, `Fraud detection model for xFusionCorp transactions.`
3. Save.

##### 6. Assign the `challenger` alias to version 1.
1. From the `fraud-detector` page, open **Version 1**.
2. At the top of the version page, find the **Aliases** field (next to Registered At / Last Modified / Source Run) and click its edit pencil.
3. Add the value `challenger` and save.

##### 7. Assign the `champion` alias to version 2.
1. Open **Version 2**.
2. Click the pencil next to the **Aliases** field at the top of the page.
3. Add the value `champion` and save.

##### 8. Verify.
**Model registry** → **fraud-detector** now shows a single registered model with two versions. The description is visible at the top of the page. Version 1 shows `@ challenger` in its Aliases field and version 2 shows `@ champion`.

> **Cross-check from the terminal (optional):**
> ```
> python3 -c "
> from mlflow import MlflowClient
> c = MlflowClient('http://localhost:5000')
> m = c.get_registered_model('fraud-detector')
> print('description:', m.description)
> for alias in ('challenger', 'champion'):
>     v = c.get_model_version_by_alias('fraud-detector', alias)
>     print(f'{alias} -> version {v.version}')
> "
> ```
> The output prints the description text, `challenger -> version 1`, and `champion -> version 2`.

#### References
- MLflow Model Registry — registered models, versions, aliases: https://mlflow.org/docs/latest/model-registry.html
- `MlflowClient` registry methods (`get_registered_model`, `get_model_version_by_alias`): https://mlflow.org/docs/latest/python_api/mlflow.client.html
