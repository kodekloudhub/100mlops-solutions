# Solution

A canary rollout ships a new model version to a small slice of traffic first, watches its error rate, and only then ramps to 100% — rolling back automatically if the candidate misbehaves. This task authors that policy in a rollout simulator: the phase weight schedule (95/5 → 70/30 → 0/100) and the rollback threshold that halts a bad version.

> As an MLOps engineer, you de-risk a model update by ramping traffic behind an automated health gate rather than flipping 100% at once — the ~5% first slice mirrors what Argo Rollouts / Flagger use. The models and data are synthetic.

#### Follow the steps below

##### 1. Confirm the starting state.
From a VS Code terminal:
```
cd /root/code/serving
cat canary_deploy.py
python3 canary_deploy.py
```
The `promote()` phase-weight ramp is a `# TODO` (so every phase keeps the initial `v1=1.0, v2=0.0`) and `ROLLBACK_THRESHOLD` ships at `1.0` — the canary policy is unwritten.

##### 2. Author the canary policy.
Open `/root/code/serving/canary_deploy.py` in the VS Code editor. Set the rollback bar to the 5 % standard:
```python
ROLLBACK_THRESHOLD = 0.05
```
Then author the phase-weight ramp in `promote()` — keep v1 the majority until v2 has proven itself, then hand v2 all traffic:
```python
    def promote(self) -> tuple[float, float]:
        """Advance to the next phase's traffic weights."""
        self.phase += 1
        if self.phase == 1:
            self.v1_weight = 0.95
            self.v2_weight = 0.05
        elif self.phase == 2:
            self.v1_weight = 0.70
            self.v2_weight = 0.30
        elif self.phase == 3:
            self.v1_weight = 0.0
            self.v2_weight = 1.0
        return self.v1_weight, self.v2_weight
```
Save the file.

##### 3. Run the simulator.
```
python3 canary_deploy.py
```
The phase-2 line now reads `Phase 2: v1=70% v2=30%`, the stats below it show `v1_requests≈70` and `v2_requests≈30`, and the run finishes with `OUTCOME: PROMOTED` + `Total requests: 300`. The healthy 2 % simulated v2 error rate stays safely below the new 5 % rollback threshold, so no rollback triggers.

##### 4. Optional — confirm the rollback path.
Bump the simulated v2 error rate temporarily to prove the rollback guard now fires:
```
python3 -c "
import canary_deploy
canary_deploy.V2_ERROR_RATE = 0.10
canary_deploy.main()
" 2>/dev/null || python3 << 'EOF'
import sys; sys.path.insert(0, '/root/code/serving')
import canary_deploy
canary_deploy.V2_ERROR_RATE = 0.10
canary_deploy.main()
EOF
```
With v2 errors at 10 %, Phase 1 or Phase 2 trips the 5 % threshold and the output reports `OUTCOME: ROLLED_BACK` — the rollback guard is now doing real work.

#### References

- Argo Rollouts — canary strategy (the phased traffic ramp this simulates): https://argo-rollouts.readthedocs.io/en/stable/features/canary/
- Argo Rollouts analysis & automated rollback (the error-rate guard): https://argo-rollouts.readthedocs.io/en/stable/features/analysis/
- Python `random.Random` — the seeded RNG driving the simulated traffic: https://docs.python.org/3/library/random.html#random.Random
