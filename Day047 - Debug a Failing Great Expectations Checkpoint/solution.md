# Solution

A **checkpoint** runs an expectation suite against a batch and records the result as a **validation run** on **Data Docs**. When a batch violates the contract the run goes **red**, and its Data Docs page is the debug surface — it names the failing expectation, the column, the bound it broke, and the observed values. From that evidence the operator decides whether the data genuinely drifted (widen the rule) or is broken (fix upstream). This task reads a failing `drift_check` run, widens the offending bound so the expectation admits the observed values, re-runs the checkpoint, and confirms Data Docs goes green.

> As an MLOps engineer, you run a validation checkpoint and repair the failing expectation from its Data Docs evidence—you are not modelling statistical concept drift. The data is synthetic.

#### Follow the steps below

##### 1. Read the failing validation on Data Docs.
Click the **Data Docs** button at the top of the lab (port `8081`). On the landing page:
- Under **Validation Results** there are two runs: one green (`default`, clean batch) and one red (`drift_check`, drifted batch).
- Click the red `drift_check` row.

The **Overview** reports `Status: ✗ Failed` — 4 evaluated, 3 successful, 1 unsuccessful (75%). The `hour`, `is_fraud`, and column-set expectations are green; the one red row is the **`amount`** expectation:

```
amount  ·  values must be greater than or equal to 0.0
  12 unexpected values found.  ≈13.04% of 92 total rows.
  Observed Value: ≈13.043% unexpected
  Unexpected values: -347.22, -279.19, -268.55, -263.9, -244.81, -240.1, … (all negative)
```

The message translates to: 12 of the 92 rows (~13%) carry a negative `amount` — the most negative is `-347.22`. The existing rule (`min_value=0`) rejects them.

##### 2. Diagnose.
From a VS Code terminal, inspect the batch directly:
```
python3 -c "
import pandas as pd
df = pd.read_csv('/root/code/dataquality/data/transactions_drifted.csv')
print(df['amount'].describe())
print('negative rows:', (df['amount'] < 0).sum())
"
```
The negative rows are legitimate refunds (decision from product: refunds are now captured in the same table as charges, as negative amounts). Tightening the upstream loader is the wrong fix — the schema itself needs to change.

##### 3. Widen the `amount` lower bound.
Open `/root/code/dataquality/fix_drift.py` in the VS Code editor. Change the `amount` expectation's `min_value` from `0` to `-5000` (or remove it entirely):

```python
    suite.add_expectation(
        ge.ExpectColumnValuesToBeBetween(column="amount", min_value=-5000)
    )
```

Save. Leave the other three expectations alone.

##### 4. Re-run the checkpoint.
```
python3 /root/code/dataquality/fix_drift.py
```

Expected output:
```
Persisted 4 expectations to `fraud_schema`
Checkpoint `drift_check` result: success=True
```

##### 5. Refresh Data Docs.
Reload the browser tab on port `8081`. The `drift_check` row is now green; clicking into the latest run shows every expectation pill reading `Success: True`.

##### 6. Verify from the terminal.
```
# Amount lower bound has been widened:
python3 -c "
import json
body = json.load(open('/root/code/dataquality/gx/expectations/fraud_schema.json'))
for e in body['expectations']:
    k = e.get('kwargs', {})
    if e.get('type') == 'expect_column_values_to_be_between' and k.get('column') == 'amount':
        print('amount min_value =', k.get('min_value'))
"

# Latest drift_check run reports success:
python3 -c "
import json, pathlib
val_dir = pathlib.Path('/root/code/dataquality/gx/uncommitted/validations')
latest = max(val_dir.rglob('*.json'), key=lambda p: p.stat().st_mtime)
body = json.loads(latest.read_text())
print('file   :', latest.name)
print('check  :', body.get('meta', {}).get('checkpoint_name'))
print('success:', body.get('success'))
"
```
The first prints `amount min_value = -5000` (or `None` if you removed the key). The second shows `success: True` for the `drift_check` run.

#### References

- `ExpectColumnValuesToBeBetween` — the `min_value` / `max_value` bounds you widen: https://greatexpectations.io/expectations/expect_column_values_to_be_between
- Expectation Gallery — every expectation and its keyword arguments: https://greatexpectations.io/expectations/
