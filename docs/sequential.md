# Sequential Stopping

Standard Bayesian A/B testing runs for a fixed duration and analyses results once. Sequential stopping evaluates evidence at periodic checkpoints and stops as soon as the evidence is strong enough — or declares futility when no meaningful winner is expected even with more data.

Frequentist peeking inflates false positive rates because p-values do not have a consistent interpretation at different sample sizes. Expected-loss stopping does not have this problem. The stopping criterion is defined in terms of a posterior quantity that has a consistent meaning at every sample size, so checking early does not bias the final result.

## StoppingChecker

`StoppingChecker` is the stateful class for experiments you check periodically. It maintains a trajectory of checkpoint snapshots and increments an internal counter at each call to `update()`.

```python
from argonx.sequential import StoppingChecker

checker = StoppingChecker(
    loss_threshold=0.01,
    prob_best_min=0.95,
    min_sample_size=1000,
)
```

### Constructor Parameters

**loss_threshold** : `float`, range `(0, 1)`, default `0.01`

The expected loss must drop below this value before stopping is permitted. This is the primary stopping criterion. Set it to reflect the business cost of being wrong — a lower threshold requires tighter evidence before the experiment stops.

**prob_best_min** : `float`, range `(0, 1)`, default `0.80`

Minimum P(best) required for stopping. Both the loss gate and the P(best) gate must pass simultaneously. The P(best) gate prevents stopping when the loss is low only because the variants are close together (which can happen with small effects and noisy data).

**min_sample_size** : `int`, default `1000`

Minimum total users (across all variants) before stopping is considered. This is a statistical power floor — the posterior simply cannot be reliable before a minimum volume of data exists, regardless of what the metrics say.

**burn_in_users** : `int`, default `500`

Minimum users per variant before the stopping machinery even begins tracking. During burn-in, posterior metrics are computed but the stopping decision is always blocked. This is a data quality floor separate from the statistical power floor. Must be less than or equal to `min_sample_size`.

**min_checkpoints** : `int`, default `3`

The stopping decision is blocked until this many checkpoints have been evaluated, regardless of what the metrics show. This prevents stopping on the first checkpoint when everything looks good by chance.

**rope_bounds** : `tuple[float, float]`, default `(-0.01, 0.01)`

Region of practical equivalence used for futility detection. If all variants show high probability of being practically equivalent (all lifts fall inside the ROPE with probability above `futility_rope_threshold`), the experiment is declared futile.

**futility_rope_threshold** : `float`, range `(0, 1)`, default `0.80`

Minimum probability that all non-control variants are inside the ROPE before declaring futility. A high value (`0.80`) means 80% of the posterior mass must be in the "no meaningful difference" region before futility fires.

**expected_traffic_shares** : `dict[str, float]`, optional

The expected traffic allocation fraction per variant. If not provided, equal allocation is assumed. Used to detect traffic imbalance. Values should sum to 1.0 — if they do not, they are normalised automatically with a warning.

```python
expected_traffic_shares={"control": 0.5, "variant_b": 0.5}
```

**imbalance_tolerance** : `float`, range `(0, 1)`, default `0.10`

Maximum allowed deviation from the expected traffic share before a variant is flagged as imbalanced. At `0.10`, a variant that receives 45% of traffic when 50% is expected (a deviation of 0.05) does not trigger the flag. A variant receiving 35% would.

**imbalance_blocks_stopping** : `bool`, default `True`

When `True`, traffic imbalance blocks stopping even if the loss and P(best) gates pass. Set to `False` if you want imbalance to generate a warning without preventing a stop decision.

**daily_traffic_per_variant** : `dict[str, float]`, optional

Estimated daily user traffic per variant. When provided, the `users_needed` estimate in the result includes a `days_to_completion` field alongside the raw user count.

```python
daily_traffic_per_variant={"control": 500.0, "variant_b": 500.0}
```

**novelty_warning_days** : `int`, default `14`

If `experiment_age_days` is passed to `update()` and is below this value, a novelty warning is added to the result and the recommendation. Novelty effects — users behaving differently because the interface is new — can inflate treatment metrics during the first one to two weeks.

**users_estimate_floor** : `int`, default `100`

The minimum additional user estimate returned when the experiment has not yet stopped. Prevents returning an estimate of zero or a trivially small number.

**users_estimate_safety_factor** : `float`, range `[1.0, ∞)`, default `1.25`

Multiplier applied to the raw users-needed projection. The projection uses a `1/sqrt(n)` posterior contraction heuristic and assumes the current effect size holds. The safety factor accounts for the fact that effect sizes are noisy at early stages. Values below `1.0` are rejected.

**min_draws_warning** : `int`, default `500`

If the posterior sample array has fewer than this many draws, a warning is emitted. Stopping decisions on thin posteriors are unreliable.

### update()

```python
checker.update(
    samples,
    variant_names,
    control,
    n_users_per_variant,
    experiment_age_days=None,
)
```

Evaluate one checkpoint and return a `StoppingResult`. The internal checkpoint counter increments with each call.

**samples** : `np.ndarray`, shape `(n_draws, n_variants)`

Posterior samples for the current checkpoint. Generate these by calling `experiment.run().samples` after fitting the model on the data collected so far.

**variant_names** : `list[str]`

Ordered list of variant names corresponding to the columns in `samples`.

**control** : `str`

The control variant name.

**n_users_per_variant** : `dict[str, int]`

Number of users observed for each variant at this checkpoint. Every name in `variant_names` must appear as a key.

**experiment_age_days** : `float`, optional

Age of the experiment in days. Used for the novelty warning check.

### Properties

**trajectory** : `list[CheckpointSnapshot]`

A read-only list of all checkpoints evaluated so far, in order. Each snapshot records the full state of the stopping gates and the posterior metrics at that point in time.

**n_checkpoints** : `int`

Number of checkpoints evaluated so far.

### plot_trajectory()

```python
checker.plot_trajectory()
```

Renders a multi-panel plot showing P(best) and expected loss over the checkpoint trajectory, with the stopping thresholds drawn as horizontal reference lines. Stopping events are marked on the time axis.

## evaluate_stopping()

`evaluate_stopping` is the stateless function version. Use it when you are managing checkpoint state yourself or running a one-off evaluation.

```python
from argonx.sequential import evaluate_stopping

result = evaluate_stopping(
    samples=samples,
    variant_names=["control", "variant_b"],
    control="control",
    n_users_per_variant={"control": 1200, "variant_b": 1195},
    loss_threshold=0.01,
    prob_best_min=0.95,
    min_sample_size=1000,
    checkpoint_index=5,
    prior_trajectory=previous_snapshots,
)
```

All parameters are the same as the `StoppingChecker` constructor plus:

**checkpoint_index** : `int`, default `1`

The current checkpoint number. Used for the `min_checkpoints` gate. When using `StoppingChecker`, this is managed automatically.

**prior_trajectory** : `list[CheckpointSnapshot]`, optional

Previous checkpoint snapshots to prepend to the trajectory in the returned result. Allows building up a trajectory across multiple calls when using the stateless function.

## StoppingResult

The object returned by both `update()` and `evaluate_stopping()`.

**safe_to_stop** : `bool`

Whether the experiment can be stopped at this checkpoint. `True` if either the winner gate or the futility gate fired.

**stopping_reason** : `"winner"` | `"futility"` | `"none"`

What triggered the stop. `"winner"` means the loss and P(best) gates both passed. `"futility"` means the ROPE-based futility check fired. `"none"` means the experiment should continue.

**best_variant** : `str`

The variant with the highest P(best) at this checkpoint.

**expected_loss** : `dict[str, float]`

Expected loss per variant at this checkpoint.

**prob_best** : `dict[str, float]`

P(best) per variant at this checkpoint.

**loss_threshold** : `float`

The threshold that was configured, for reference.

**gate_states** : `dict[str, bool]`

Whether each gate has passed, keyed by gate name: `"burn_in"`, `"sample_size"`, `"min_checkpoints"`, `"loss"`, `"prob_best"`, `"traffic"`. Useful for diagnosing why the experiment has not stopped.

**traffic** : `TrafficDiagnostics`

Traffic balance information for this checkpoint: observed shares per variant, expected shares, maximum deviation, and which variants are flagged.

**users_needed** : `UsersNeededEstimate` or `None`

Estimate of additional users needed before stopping is expected to be possible. `None` when the experiment is already past the prerequisites and the remaining blocker is the loss or P(best) gate not yet clearing. The estimate uses `1/sqrt(n)` posterior contraction with the configured safety factor.

**novelty_warning** : `bool`

Whether the experiment age is below `novelty_warning_days`.

**futility_triggered** : `bool`

Whether the futility check fired at this checkpoint.

**recommendation** : `str`

Plain-English recommendation string, suitable for display. Includes which gates are blocking, the current loss and P(best) values, and the users-needed estimate if applicable.

**checkpoint_index** : `int`

The checkpoint number this result corresponds to.

**trajectory** : `list[CheckpointSnapshot]`

All checkpoints up to and including this one.

**warnings** : `list[str]`

Warnings emitted during evaluation (thin posterior, traffic imbalance, novelty).

## The Gate Sequence

Stopping only fires when all prerequisites are met and the evidence criteria are satisfied. The gates are evaluated in order:

1. **Burn-in gate**: every variant must have at least `burn_in_users` observations. Blocks during initial data collection.
2. **Sample size gate**: total users across all variants must reach `min_sample_size`. Statistical power floor.
3. **Checkpoint gate**: at least `min_checkpoints` evaluations must have occurred. Prevents stopping on an early run of good luck.
4. **Traffic gate**: traffic allocation must be within `imbalance_tolerance` of the expected shares (if `imbalance_blocks_stopping=True`).

Once all prerequisites pass, the evidence gates are checked:

5. **Loss gate**: `expected_loss[best_variant] < loss_threshold`.
6. **P(best) gate**: `prob_best[best_variant] >= prob_best_min`.

Separately, the futility gate fires if all prerequisites pass and the ROPE check indicates no meaningful winner is expected.

The `gate_states` dict in `StoppingResult` tells you exactly which gates have and have not passed, so you can diagnose why the experiment is still running.

## Practical Patterns

**Typical weekly-checkpoint experiment**

```python
checker = StoppingChecker(
    loss_threshold=0.01,
    prob_best_min=0.95,
    min_sample_size=2000,
    min_checkpoints=2,
    daily_traffic_per_variant={"control": 300.0, "variant_b": 300.0},
)

for week in range(1, 9):
    # fit model on data collected so far
    result = experiment.run(n_draws=2000)

    status = checker.update(
        samples=result.samples,
        variant_names=["control", "variant_b"],
        control="control",
        n_users_per_variant=get_user_counts(week),
        experiment_age_days=week * 7,
    )

    print(f"Week {week}: {status.recommendation}")

    if status.safe_to_stop:
        print(f"Stopping: {status.stopping_reason}")
        break
```

**One-off evaluation without state**

```python
from argonx.sequential import evaluate_stopping

status = evaluate_stopping(
    samples=result.samples,
    variant_names=["control", "variant_b"],
    control="control",
    n_users_per_variant={"control": 5000, "variant_b": 4980},
    checkpoint_index=1,
)
```
