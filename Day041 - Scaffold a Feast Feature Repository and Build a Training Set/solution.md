# Solution

A **feature store** like Feast is the shared layer between feature engineering and model serving: it keeps feature *definitions* (entities, feature views, schemas) in a versioned **feature repository** + **registry**, so training and serving read the *same* features. `feast init` scaffolds a repo, `feast apply` registers its definitions, and `feast ui` serves a read-only dashboard. The store then serves features two ways: the **offline** path — `get_historical_features` — builds **training** data with a *point-in-time* join (each row sees a feature's value *as of* its event timestamp, so no future data leaks into training); the **online** path serves the latest values at inference (covered later). This task scaffolds the repo, builds a point-in-time training set from the offline store, and opens the UI.

> As an MLOps engineer, you scaffold a shared feature store so training and serving read the exact same feature values — you are not engineering new features; the data is synthetic.

#### Follow the steps below

##### 1. Confirm the starting state.
From a VS Code terminal, verify the Feast CLI is available and the target directory is empty:
```
feast version
ls /root/code/
```
The `feast version` output prints the installed Feast version. `/root/code/` exists but has no `feature_repo/` yet.

##### 2. Initialise the feature repository.
`feast init` scaffolds a complete starter project — `feature_store.yaml`, `feature_definitions.py`, a sample dataset, and a local SQLite-backed online store:
```
cd /root/code
feast init feature_repo
```

##### 3. Apply the starter definitions to the registry.
`feast apply` reads the example repo and writes the registry + schema files. The command runs from the inner `feature_repo/` directory where `feature_store.yaml` lives:
```
cd /root/code/feature_repo/feature_repo
feast apply
```

##### 4. Inspect the scaffold.
```
cat feature_store.yaml
ls data/
```
`feature_store.yaml` carries `project`, `provider`, `registry`, `online_store`, and `offline_store` keys. The `data/` directory now contains `registry.db` — the proof that `feast apply` completed.

##### 5. Build a point-in-time training set.
The store's offline path turns feature definitions into training data. Open `/root/code/build_training_set.py` — it reads `(driver_id, event_timestamp)` rows from the scaffold's source; complete the TODO to join the features as of each timestamp:
```python
training_df = store.get_historical_features(
    entity_df=entity_df,
    features=[
        "driver_hourly_stats:conv_rate",
        "driver_hourly_stats:acc_rate",
        "driver_hourly_stats:avg_daily_trips",
    ],
).to_df()
```
Run it (the `FeatureStore(repo_path=…)` inside points at the applied repo):
```
python3 /root/code/build_training_set.py
```
It writes `/root/code/training_set.parquet` — the entity rows now carry `conv_rate`, `acc_rate`, and `avg_daily_trips` joined *as of* each event timestamp. That point-in-time join is what keeps training honest (a row never sees a feature value from after its own timestamp).

##### 6. Start the Feast UI.
The Feast UI is a read-only dashboard over the registry. It must run from a feature_repo directory. Start it in the background so the terminal stays usable:
```
feast ui --host 0.0.0.0 --port 8888 &
```
Wait a few seconds for the server to bind port `8888`.

##### 7. Verify in the Feast UI.
Open the **Feast UI** button at the top of the lab. The dashboard lists the scaffold's project (`feature_repo`) with **Data Sources (3)**, **Entities (2)** — `driver` plus Feast's built-in `__dummy` entity (used by on-demand views) — **Features (12)**, **Feature Views (4)** (`driver_hourly_stats` and `driver_hourly_stats_fresh`, plus the on-demand `transformed_conv_rate` and `transformed_conv_rate_fresh`), and **Feature Services (3)** (`driver_activity_v1` through `driver_activity_v3`). The **Lineage** tab renders the source → entity → view → service graph. The UI loading cleanly confirms the registry is valid and Feast is wired end-to-end. (The entities show **Value Type** `INVALID` — this is cosmetic: the stock `feast init` definitions don't set an entity `value_type`, so Feast infers the type from the source. It doesn't affect the training-set build.)

#### References
- Feast quickstart (`feast init` / `feast apply` / `feast ui`): https://docs.feast.dev/getting-started/quickstart
- Feast concepts — feature repository, registry, feature store: https://docs.feast.dev/getting-started/concepts
- Feast CLI commands reference: https://docs.feast.dev/reference/feast-cli-commands
- Feast feature retrieval — `get_historical_features` (point-in-time training data): https://docs.feast.dev/getting-started/concepts/feature-retrieval
