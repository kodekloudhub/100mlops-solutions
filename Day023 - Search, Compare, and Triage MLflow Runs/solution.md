# Solution

Once an experiment holds many runs, picking winners and losers by eye stops working. Three MLflow tools make it tractable: **search** filters runs with a small query language — comparisons on `metrics.*`, `params.*`, and `tags.*` (for example `metrics.f1_score > 0.85`) typed straight into the run-list **Search** bar (the same syntax the `mlflow.search_runs()` API uses); the **Compare** view lines selected runs up side by side (a metric table plus a parallel-coordinates plot) so the best on a chosen metric is obvious; and **run-level tags** (key/value pairs such as `review-status=shortlisted`) record the triage decision on each run so it persists and is itself searchable. This task combines all three — search to narrow, compare to confirm, tag to record — shortlisting the best run and rejecting the under-performers.

> As an MLOps engineer, you triage a crowded experiment by searching, comparing, and tagging runs so promotion decisions are recorded and reproducible — you are not judging model quality; the seeded metrics are synthetic.

#### Follow the steps below

Open the MLflow UI via the **MLflow UI** button at the top of the lab.

##### 1. Open the `fraud-detection` experiment.
From the Experiments sidebar, click `fraud-detection`. The run-list view opens, displaying the ten pre-seeded runs with their `n_estimators`, `max_depth`, `f1_score`, and `accuracy`.

##### 2. Compare the candidates side by side.
1. Select several runs with the checkboxes in the leftmost column (the header checkbox selects every visible row), then click **Compare** at the top of the run list.
2. The Compare view shows a parameter table, a side-by-side metric table, and a parallel-coordinates plot. Read across the `f1_score` row (or the `f1_score` axis of the plot) to see how the candidates rank — this is how you confirm which run is strongest before tagging it. Return to the run list when done.

##### 3. Shortlist the best candidate.
1. In the **Search** bar above the run list, enter the filter:
   ```
   metrics.f1_score > 0.85
   ```
   Submit the filter — the run list narrows to the five runs that meet the threshold.
2. Click the `f1_score` column header to sort the visible rows in descending order. The run with `f1_score = 0.95` moves to the top of the table.
3. Click that top run's name to open its detail page.
4. Open the run's **Tags** panel and use the **Add tag** control:
   - **Key**: `review-status`
   - **Value**: `shortlisted`
5. Save the tag and return to the experiment page.

##### 4. Reject the under-performing runs.
1. Replace the current filter with:
   ```
   metrics.f1_score < 0.75
   ```
   The run list narrows to two runs (those with `f1_score = 0.72` and `0.70`).
2. Select both rows using the checkboxes in the leftmost column.
3. Apply a tag to the selection. If the UI exposes a bulk-tag action in the top toolbar, use it with **Key** `review-status` and **Value** `rejected`. Otherwise, open each of the two runs in turn and add the same tag on its Tags panel.
4. Clear the search bar so the full run list is visible again.

##### 5. Verify.
Open each relevant run and confirm:
- The run with `f1_score = 0.95` carries the tag `review-status = shortlisted`.
- The two runs with `f1_score = 0.72` and `f1_score = 0.70` both carry the tag `review-status = rejected`.
- The remaining seven runs carry no `review-status` tag.

> **Cross-check from the terminal (optional):**
> ```
> python3 -c "
> from mlflow import MlflowClient
> c = MlflowClient('http://localhost:5000')
> exp = c.get_experiment_by_name('fraud-detection')
> for r in c.search_runs([exp.experiment_id]):
>     tag = r.data.tags.get('review-status', '-')
>     f1 = r.data.metrics.get('f1_score')
>     print(f'{r.info.run_id[:8]}  f1={f1:.2f}  review-status={tag}')
> "
> ```
> The shortlisted run's `f1_score` is the largest; each rejected run has `f1_score < 0.75`; every other run shows `review-status=-`.

#### References
- MLflow Tracking UI — comparing runs side by side (the Compare view): https://mlflow.org/docs/latest/tracking.html
- MLflow search syntax — filtering runs (`metrics.*`, `params.*`, `tags.*`): https://mlflow.org/docs/latest/search-runs.html
- `mlflow.search_runs` (same filter language, programmatic): https://mlflow.org/docs/latest/python_api/mlflow.html
- `MlflowClient` (used in the optional terminal cross-check): https://mlflow.org/docs/latest/python_api/mlflow.client.html
