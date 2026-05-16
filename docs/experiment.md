# Experiment

`Experiment` is the top-level class. It validates inputs at construction time and defers all inference to `run()`.

```python
from argonx import Experiment
```

## Constructor

```python
Experiment(
    data,
    variant_col,
    primary_metric,
    model,
    guardrails=None,
    lower_is_better=None,
    segment_col=None,
    control=None,
    priors=None,
    guardrail_models=None,
)
```

### Parameters

**data** : `pd.DataFrame`

The full experiment dataset. One row per observation, with at minimum a column identifying the variant. The frame is not copied — avoid mutating it after construction.

**variant_col** : `str`

Name of the column that identifies which variant each row belongs to. Must exist in `data`.

**primary_metric** : `str` or `callable`

The metric being optimised. Pass a column name as a string, or a callable that accepts a `pd.DataFrame` and returns a `pd.Series`. Use the callable form for ratio metrics or any derived quantity that does not exist as a single column.

```python
# column name
primary_metric="revenue"

# derived ratio
primary_metric=lambda df: df["clicks"] / df["impressions"]
```

**model** : `str`

Which generative model to use for the primary metric. One of `"binary"`, `"lognormal"`, `"gaussian"`, `"studentt"`, `"poisson"`. See [models.md](models.md) for guidance on which to choose.

**guardrails** : `list[str]`, optional

Column names of secondary metrics that must not degrade. The engine evaluates each guardrail independently and reports a conflict when the primary metric improves but a guardrail degrades. Defaults to no guardrails.

**lower_is_better** : `dict[str, bool]`, optional

For each guardrail where a lower value is better (latency, error rate, churn), set the entry to `True`. Any guardrail not listed here is assumed to be higher-is-better. Keys must be a subset of `guardrails`.

```python
lower_is_better={"page_load_ms": True, "error_rate": True}
```

**segment_col** : `str`, optional

Column name used to split the data into segments for hierarchical inference. When set, the experiment automatically uses the hierarchical variant of the selected model. Each segment gets its own posterior, connected through a shared hyperprior that pools information across segments proportionally to sample size.

**control** : `str`, optional

The name of the control variant within `variant_col`. Used as the baseline for lift computation, ROPE, and expected loss. If not provided, the lexicographically first variant is used as control.

**priors** : `dict`, optional

Model-specific prior overrides for hierarchical experiments. The structure depends on the model. See [models.md](models.md) for what each model accepts.

**guardrail_models** : `dict[str, str]`, optional

Model override per guardrail. Keys are guardrail column names, values are model strings from the same registry as `model`. Any guardrail not listed here uses the primary model as fallback.

```python
guardrail_models={
    "page_load_ms": "gaussian",
    "error_rate": "binary",
}
```

## run()

```python
Experiment.run(
    min_effect=0.01,
    rope_bounds=None,
    composite_weights=None,
    n_draws=2000,
    random_seed=None,
    config=None,
)
```

Runs inference and returns a `Results` object.

### Parameters

**min_effect** : `float`, optional

The minimum relative lift that counts as practically meaningful. Defines the symmetric ROPE as `(-min_effect, min_effect)`. Ignored if `rope_bounds` is passed directly. Default `0.01` (1%).

**rope_bounds** : `tuple[float, float]`, optional

Explicit ROPE bounds as `(low, high)`. Use this when the practically irrelevant region is not symmetric. Overrides `min_effect` when provided.

**composite_weights** : `dict[str, float]`, optional

Weights for composite decision scoring. Keys are metric names (primary metric name or guardrail column names). The composite score is computed draw-by-draw from posteriors, so uncertainty propagates into the final score rather than being hidden in point estimates.

```python
composite_weights={"revenue": 0.7, "page_load_ms": 0.3}
```

**n_draws** : `int`, optional

Number of posterior draws. For conjugate models (binary, poisson), this is exact sampling and has no meaningful accuracy tradeoff. For MCMC models (lognormal, gaussian, studentt), more draws narrow the Monte Carlo error on CVaR and HDI estimates. Default `2000`. Minimum recommended for reliable CVaR is `1000`.

**random_seed** : `int`, optional

Passed to the model sampler for reproducibility. Has no effect on conjugate models whose posteriors are exact.

**config** : `dict`, optional

Overrides for engine decision parameters. See the Config Reference section below.

### Returns

**result** : `Results`

See [results.md](results.md) for the full interface.

## Config Reference

The `config` dict is validated at call time. Unknown keys raise `ValueError`. All keys are optional — only pass what you want to override.

### Decision Thresholds

**prob_best_strong** : `float`, range `(0, 1)`, default `0.95`

Minimum P(best) required for the primary signal to be classified as `"strong"`. Strong is needed for a `"strong win"` verdict. Raise this to require more certainty before recommending a ship.

**prob_best_moderate** : `float`, range `(0, 1)`, default `0.80`

Minimum P(best) for the signal to be classified as `"moderate"`. A moderate signal with low risk and practical significance yields a `"weak win"` verdict.

**expected_loss_max** : `float`, range `[0, ∞)`, default `0.01`

Maximum acceptable expected loss for a clean decision. Expected loss is the average posterior loss from selecting the wrong variant — it integrates the full distribution rather than just using a point estimate. Smaller values require tighter evidence.

**cvar_ratio_max** : `float`, range `[1, ∞)`, default `5.0`

Maximum acceptable ratio of CVaR to expected loss. A high ratio means the tail risk is disproportionate even if the average loss looks fine. When `CVaR / expected_loss > cvar_ratio_max`, the risk classification escalates even if `expected_loss` is below `expected_loss_max`.

**rope_practical_min** : `float`, range `(0, 1)`, default `0.80`

Minimum posterior probability that the effect is outside the ROPE for the decision to count as practically significant. The ROPE check answers "is this effect large enough to matter?" independent of statistical confidence.

### Inference Parameters

**alpha** : `float`, range `(0, 1)`, default `0.95`

Confidence level for CVaR computation. At `alpha=0.95`, CVaR is the expected loss in the worst 5% of posterior draws. Determines the tail size.

**hdi_prob** : `float`, range `(0, 1)`, default `0.95`

Width of the Highest Density Interval reported for lift. The HDI is the shortest interval containing this fraction of posterior mass. It is not a frequentist confidence interval — it is a direct probability statement about where the lift is.

### Guardrail Parameters

**guardrail_thresholds** : `dict[str, float]`, default `{}`

Per-metric custom thresholds for guardrail failure. The threshold is the maximum tolerable P(degraded) before a guardrail is marked as failed. If a metric is not listed, the default threshold of `0.10` is used.

```python
config={
    "guardrail_thresholds": {
        "page_load_ms": 0.05,   # fail if P(degraded) > 5%
        "error_rate": 0.15,      # more lenient for error rate
    }
}
```

**guardrail_penalty** : `float`, range `[0, ∞)`, default `0.0`

Penalty subtracted from the composite score for each failed guardrail. Only relevant when `composite_weights` is used. Set to a positive value to make guardrail failures directly reduce the composite ranking.

**deterioration_weights** : `dict[str, float]`, optional

Per-metric weights for the deterioration term in the composite score calculation. Controls how much each guardrail degradation penalises the composite score independently of `guardrail_penalty`.

### Joint and Composite

**metrics_to_join** : `list[str]`, optional

The subset of metrics to include in the joint probability calculation. By default, when guardrails are present, all metrics are joined. Restrict this to focus the joint probability on the metrics that matter most for your policy.

**composite_threshold** : `float`, optional

The score threshold that a variant's composite score must exceed to be considered a positive decision. Defaults to `0.0` when composite scoring is active.

**primary_lower_is_better** : `bool`, default `False`

Set to `True` when the primary metric should be minimised rather than maximised (e.g., latency, churn rate). This changes the direction of the lift computation and the joint probability calculation for the primary metric.

**lower_is_better** : `dict[str, bool]`, default `{}`

Can also be set through config in addition to the constructor argument. If the same key appears in both, the constructor value takes precedence and a warning is emitted.

## repr

Calling `repr()` or printing an `Experiment` object shows a summary of what was configured:

```
Experiment(model=lognormal, variants=['control', 'variant_b'], control=control)
Experiment(model=lognormal, variants=['control', 'variant_b'], control=control, segments=['desktop', 'mobile'])
```
