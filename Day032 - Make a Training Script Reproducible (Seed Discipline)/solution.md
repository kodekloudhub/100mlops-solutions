# Solution

**Reproducibility** means the same code on the same data produces the same result every run — the foundation for checksum-based caching, fair experiment comparison, and audit trails. The enemy is uncontrolled randomness: scikit-learn's randomised operations each draw from a random stream, and unless you **pin the seed** (`random_state=42`) on every one of them, each run samples differently and the metrics drift. The discipline is to find *every* random stream and seed it — here there are two: the train/test **split** and the **RandomForest** (its bootstrap sampling and per-split feature selection). This task seeds both so a 3-run probe gets byte-identical metrics.

> As an MLOps engineer, you make a training run deterministic and byte-for-byte repeatable by controlling its seeds—you are not chasing better metric values. The data is synthetic and the metric numbers carry no meaning.

#### Follow the steps below

##### 1. Confirm the starting state.
Open the **MLflow UI** button at the top of the lab — the `fraud-detection-repro` experiment is present and empty. From a VS Code terminal:
```
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:5000/
```
A `200` confirms the MLflow server is reachable.

##### 2. Run the determinism probe and observe the failure.
```
/root/code/fraud-detection/check_determinism.sh
```
The probe runs the trainer three times and `diff`s each adjacent pair of metrics JSON files. Output ends with `FAIL: the three runs did not produce byte-identical metrics.` followed by unified diffs that highlight the differences in `accuracy`, `f1_score`, and `feature_importances`.

##### 3. Inspect the trainer.
Open `/root/code/fraud-detection/src/models/train.py` in the VS Code editor. Note two scikit-learn calls whose output depends on the global random state:
- `train_test_split(X, y, test_size=0.2, stratify=y)`
- `RandomForestClassifier(n_estimators=100, max_depth=5)`

Neither pins its own random state, so the sampled test split and the estimator's bootstrap / feature-sampling both change from run to run.

##### 4. Add seed discipline.
Pass `random_state=42` to both call sites:
```python
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, stratify=y, random_state=42
)

model = RandomForestClassifier(n_estimators=100, max_depth=5, random_state=42)
```
The rest of the script is already correct; save the file without touching anything else.

##### 5. Re-run the probe.
```
/root/code/fraud-detection/check_determinism.sh
```
The probe now prints `OK: all three runs produced byte-identical metrics.` and exits with status `0`.

##### 6. Verify in the MLflow UI.
Open the **MLflow UI** button → `fraud-detection-repro` experiment → the run list shows the three probe runs named `repro-run-1`, `repro-run-2`, and `repro-run-3`. Clicking between them (or using the Compare view) shows `metrics.accuracy` and `metrics.f1_score` match to six decimal places across all three.

#### References
- scikit-learn — Controlling randomness (`random_state`, reproducibility pitfalls): https://scikit-learn.org/stable/common_pitfalls.html
- `train_test_split` (`random_state` parameter): https://scikit-learn.org/stable/modules/generated/sklearn.model_selection.train_test_split.html
- `RandomForestClassifier` (`random_state` over bootstrap + feature sampling): https://scikit-learn.org/stable/modules/generated/sklearn.ensemble.RandomForestClassifier.html
