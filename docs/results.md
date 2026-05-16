# Results

`Results` is the object returned by `Experiment.run()`. It exposes the decision recommendation, all computed metrics, visualisation methods, and export methods.

```python
result = experiment.run()
```

## Quick Overview

```python
result.summary()            # full human-readable output
result.segment_summary()    # per-segment breakdown (hierarchical only)
result.plot()               # five-panel decision dashboard
result.plot_segments()      # posterior grid by segment (hierarchical only)
result.to_dict()            # fully serialisable dict
result.to_dataframe()       # per-variant metrics as a DataFrame
```

## Attribute Access

`Results` proxies attribute access to the underlying `DecisionResult` dataclass. Any field on `DecisionResult` is accessible directly on the result object:

```python
result.state                # "strong win" | "weak win" | "high risk" | "guardrail conflicts" | "inconclusive"
result.recommendation       # "ship variant" | "consider shipping" | "do not ship" | "review required" | "continue experiment"
result.best_variant         # name of the leading variant
result.primary_strength     # "strong" | "moderate" | "weak"
result.risk_level           # "low" | "medium" | "high"
result.practical_significance  # "yes" | "uncertain" | "no"
result.guardrail_status     # "pass" | "fail"
result.confidence           # "high" | "medium" | "low"
result.reasons              # list[str]: human-readable decision drivers
result.notes                # list[str]: warnings and flagged issues
```

Accessing `result.metrics` returns a `MetricsBundle`, `result.guardrails` returns a `GuardrailBundle`, `result.joint` returns a `JointResult` (or `None`), and `result.composite` returns a `CompositeResult` (or `None`).

### samples

`result.samples` is the posterior sample array of shape `(n_draws, n_variants)` for the primary metric. Pass this to `StoppingChecker.update()` or to `result.plot(samples=...)`.

For hierarchical experiments, `result.segment_samples` is the per-segment sample array of shape `(n_draws, n_segments, n_variants)`.

### segment_results

`result.segment_results` is a `dict[str, DecisionResult]` mapping segment names to their individual decision results. Only populated for hierarchical experiments. Each value has the same attribute structure as the top-level result.

### segment_guardrail_violations

`result.segment_guardrail_violations` is a `dict[str, list[str]]` mapping segment names to the list of guardrail metrics that failed in that segment. Only includes segments with at least one failure. Displayed in `result.summary()` when present, even though the aggregate result may show all guardrails passing — segment-level violations do not bubble up to the aggregate automatically.

## summary()

```python
result.summary()
```

Prints a structured text report to stdout. The sections are:

**PRIMARY METRIC** — best variant, expected lift with HDI bounds, and P(best).

**RISK** — expected loss, CVaR, and the risk level classification.

**PRACTICAL SIGNIFICANCE (ROPE)** — whether the effect is inside, outside, or straddling the ROPE, and the probability of practical significance.

**GUARDRAILS** — one line per guardrail showing pass/fail, P(degraded), the threshold used, and severity.

**GUARDRAIL CONFLICTS DETECTED** — shown only when primary signal is strong and guardrails fail. Names each conflict and states that human review is required.

**SEGMENT GUARDRAIL VIOLATIONS** — shown only for hierarchical experiments where segment-level guardrail failures exist that may not be visible in the aggregate.

**JOINT POLICY PROBABILITY** — shown when guardrails were defined. Shows the joint probability per variant, the independence benchmark, and the correlation gap.

**COMPOSITE DECISION SCORE** — shown when composite weights were passed to `run()`. Shows the weighted score per variant, P(score exceeds threshold), and the gap HDI.

**DECISION** — state, recommendation, confidence, and a list of the decision drivers. Any flagged issues appear at the bottom.

## segment_summary()

```python
result.segment_summary()
```

Prints a per-segment breakdown for hierarchical experiments. Raises `RuntimeError` if called on a flat experiment result.

For each segment, it shows the best variant, state, recommendation, expected lift, and guardrail status. Failed guardrails are expanded to show P(degraded) and severity.

After the per-segment section, a cross-segment analysis identifies:

**Inconsistent winner** — whether different segments have different best variants.

**Shipping conflict** — whether some segments support shipping and others do not. The conflicting segments are listed explicitly rather than averaged. A shipping conflict means universal rollout is not safe and a segment-targeted approach should be considered.

**Consistent signal** — when all segments agree on shipping or not shipping, this is stated plainly.

## plot()

```python
result.plot(
    samples=None,
    metric_name="metric",
    rope_bounds=None,
    figsize=(18, 11),
    suptitle=None,
)
```

Renders a five-panel decision dashboard and returns a `matplotlib.figure.Figure`.

**samples** : `np.ndarray`, optional

Posterior samples to use for the plot. Defaults to `result.samples`. Pass explicitly if you want to plot a different sample set.

**metric_name** : `str`, optional

Label for the primary metric axis. Defaults to `"metric"`.

**rope_bounds** : `tuple[float, float]`, optional

ROPE bounds to display on the ROPE panel. Defaults to the bounds used during the experiment run.

**figsize** : `tuple[int, int]`, optional

Figure size in inches. Default `(18, 11)`.

**suptitle** : `str`, optional

Title for the entire figure.

The five panels are:

1. Posterior distributions of the primary metric for each variant.
2. Lift with HDI relative to control, with the ROPE region shaded.
3. P(best) per variant as a bar chart.
4. Expected loss per variant alongside the configured threshold.
5. Guardrail P(degraded) per metric per variant, with the threshold drawn as a horizontal line.

## plot_segments()

```python
result.plot_segments(
    segment_names=None,
    figsize=None,
)
```

Generates a grid of posterior distribution plots, one panel per segment. Only available for hierarchical experiments. Raises `ValueError` if segment samples are not present.

**segment_names** : `list[str]`, optional

Subset of segments to plot. Defaults to all segments.

**figsize** : `tuple[int, int]`, optional

Figure size. Defaults to a size scaled to the number of segments.

## to_dict()

```python
d = result.to_dict()
```

Returns a fully serialisable `dict`. All numpy arrays are converted to Python lists, all numpy scalars are converted to Python floats. The output is safe for `json.dumps()`.

Top-level keys:

**decision** — state, recommendation, best_variant, confidence, primary_strength, risk_level, practical_significance, guardrail_status, reasons, notes.

**metrics** — prob_best, expected_loss, cvar, lift_mean, lift_hdi_low, lift_hdi_high, rope_inside, rope_outside, prob_practical. All as `dict[str, float]` keyed by variant name.

**guardrails** — all_passed (bool), variant_passed (dict), results (list of per-guardrail dicts with metric, variant, passed, prob_degraded, threshold, severity, expected_degradation), conflicts (list of conflict dicts with metric, variant, prob_degraded, threshold, severity, message).

**joint** — joint_prob, condition_probs, independence_benchmark, correlation_gap, best_variant, metrics_joined. `None` when no guardrails were defined.

**composite** — score, prob_exceeds_threshold, gap_hdi, metric_contributions, best_variant, threshold. `None` when composite weights were not passed.

**segment_results** — `dict[str, dict]` with per-segment state, recommendation, best_variant, prob_best, expected_loss, lift_mean, guardrail_passed, notes. `None` for flat experiments.

**segment_guardrail_violations** — `dict[str, list[str]]` or `None`.

## to_dataframe()

```python
df = result.to_dataframe()
```

Returns per-variant metrics as a `pandas.DataFrame`.

**Flat experiments**: the index is `variant`. Columns include prob_best, expected_loss, cvar, lift_mean, lift_hdi_low, lift_hdi_high, prob_practical, inside_rope, guardrail_passed, joint_prob, correlation_gap, composite_score, prob_exceeds_threshold. The control variant is excluded from the rows — the frame shows treatment variants only.

**Hierarchical experiments**: the index is a `MultiIndex` of `(segment, variant)`. The first level is `"aggregate"` for the population-level results, followed by one entry per segment. This means the dataframe has both the aggregate decision and the per-segment decisions in a single structure that is easy to filter and compare.

```python
# aggregate only
df.loc["aggregate"]

# single segment
df.loc["mobile"]

# compare lift across segments
df.xs("lift_mean", axis=1)
```

## repr

Printing a `Results` object in a notebook shows a compact one-glance summary:

```
Results(
  state          = 'strong win'
  recommendation = 'ship variant'
  best_variant   = 'variant_b'
  prob_best      = 0.971
  lift_mean      = 0.043
  guardrails     = PASS
  notes          = 0 flagged
)
```

For hierarchical experiments, the segment list is also shown.

## MetricsBundle fields

When accessing `result.metrics` directly:

```python
result.metrics.prob_best.probabilities    # dict[str, float]
result.metrics.prob_best.best_variant     # str
result.metrics.loss.expected_loss         # dict[str, float]
result.metrics.loss.loss_distributions    # dict[str, np.ndarray]
result.metrics.cvar.cvar                  # dict[str, float]
result.metrics.cvar.var_threshold         # dict[str, float]
result.metrics.cvar.alpha                 # float
result.metrics.rope.inside_rope           # dict[str, float]
result.metrics.rope.outside_rope          # dict[str, float]
result.metrics.rope.prob_practical        # dict[str, float]
result.metrics.lift.mean                  # dict[str, float]
result.metrics.lift.hdi_low               # dict[str, float]
result.metrics.lift.hdi_high              # dict[str, float]
result.metrics.lift.hdi_prob              # float
result.metrics.warnings                   # list[str]
```

## GuardrailBundle fields

```python
result.guardrails.all_passed         # bool
result.guardrails.variant_passed     # dict[str, bool]
result.guardrails.guardrails         # list[GuardrailResult]
result.guardrails.conflicts          # list[GuardrailConflict]
result.guardrails.warnings           # list[str]
```

Each `GuardrailResult` has: `metric`, `variant`, `passed`, `prob_degraded`, `threshold`, `severity`, `expected_degradation`.

Each `GuardrailConflict` has: `metric`, `variant`, `prob_degraded`, `threshold`, `severity`, `message`.
