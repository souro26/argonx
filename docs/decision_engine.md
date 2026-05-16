# Decision Engine

The decision engine takes posterior samples from the model layer and produces a structured recommendation. This document explains every metric it computes, how those metrics feed into the decision states, and what each state actually means.

## Metrics

All metrics are computed from posterior samples — arrays of shape `(n_draws, n_variants)`. Each row is one complete draw from the joint posterior. The engine never reduces this to point estimates before computing a metric; it operates on the full distribution and summarises at the end.

### P(best)

P(best) is the probability that a given variant is genuinely the best across all variants simultaneously. At each posterior draw, the variant with the highest sampled value wins that draw. P(best) is the fraction of draws each variant wins.

This is the correct N-variant formulation. Pairwise comparisons (P(A > B), P(A > C), etc.) are wrong for three or more variants because they double-count probability mass. The argmax formulation is not an approximation — it is the exact posterior probability of being the best option.

Values across all variants sum to 1.0. When the primary metric favours a single variant strongly, that variant's P(best) approaches 1. When the experiment is genuinely underpowered or the effect is small, P(best) stays close to the uniform prior `1/n_variants`.

### Expected Loss

Expected loss for variant `v` is the average additional value you give up by choosing `v` on draws where `v` is not actually the best. Formally, at each draw, the loss from choosing `v` is `max(0, best_alternative - v)`. Averaged over all draws, this is the expected regret from shipping variant `v`.

A small expected loss means that even when variant `v` loses a draw, it loses by very little — the cost of being wrong is low. A large expected loss means the cost of being wrong about variant `v` could be substantial.

The expected loss of the actual best variant is the most useful number. If the experiment's best variant has an expected loss of `0.003` and your threshold is `0.01`, the evidence is strong enough.

### CVaR

CVaR (Conditional Value at Risk, also called Expected Shortfall) is the expected loss in the worst-`alpha` fraction of posterior draws. At `alpha=0.95`, it is the average loss in the worst 5% of outcomes.

CVaR is the tail-risk complement to expected loss. A variant can have a low expected loss but a high CVaR — meaning most draws are fine but the tail outcomes are bad. The engine checks whether `CVaR / expected_loss` exceeds `cvar_ratio_max` (default 5). If it does, the risk classification escalates even if the mean loss looks acceptable.

CVaR requires enough draws in the tail to be stable. At `n_draws=1000`, the tail has about 50 observations at `alpha=0.95`. The engine warns if `n_draws < 1000`.

### ROPE

ROPE (Region of Practical Equivalence) answers whether the effect is large enough to matter in practice. It is defined as a symmetric interval `(-min_effect, min_effect)` around zero, where `min_effect` is set by the caller.

The engine computes, for each variant, the fraction of posterior lift draws that fall inside and outside the ROPE. A statistically real effect can still be practically irrelevant — if `P(lift inside ROPE) = 0.85`, the effect is probably real but probably too small to justify the engineering and operational cost of shipping.

`prob_practical` is `P(lift > rope_high)` — the probability that the lift is not just real but actually large enough to matter. This is what feeds the practical significance gate.

### HDI

HDI (Highest Density Interval) is the shortest interval containing a given fraction of posterior mass, typically 95%. It is the Bayesian analogue of a confidence interval, with the key difference that it is an actual probability statement: the true lift lies within the HDI with 95% posterior probability.

The HDI is computed for the lift distribution of each variant relative to control. Asymmetric or skewed lift distributions produce HDIs that are not centred on the mean — the HDI correctly reflects where the mass is.

### Guardrail Evaluation

Each guardrail metric is evaluated by its own model, and the engine computes P(degraded) — the posterior probability that the guardrail metric moves in the wrong direction. "Wrong direction" is determined by `lower_is_better`. For a latency metric with `lower_is_better=True`, P(degraded) is the probability that latency increases.

The guardrail fails if `P(degraded) > threshold`, where threshold defaults to `0.10` and can be overridden per metric via `config["guardrail_thresholds"]`.

A conflict is declared when the primary signal is strong (the engine is leaning toward shipping) and at least one guardrail fails. The engine does not attempt to resolve this tradeoff automatically. It reports the conflict and stops there, because the right resolution depends on business context that is not encoded in the data.

### Joint Probability

When guardrails are present, the engine computes the joint probability that all conditions hold simultaneously across the primary metric and all guardrails. This is not the product of the individual probabilities — that would be correct only if the metrics were independent, and they usually are not.

The joint probability is computed draw-by-draw: for each draw, either all conditions are met or they are not. The fraction of draws where all conditions hold is the joint probability. The `correlation_gap` field shows the difference between the joint probability and what it would be under independence (`joint_prob - product_of_individual_probs`). A negative gap means the conditions are negatively correlated — achieving one makes the others less likely — which is common when primary improvement comes at the cost of latency.

### Composite Score

When `composite_weights` is passed to `run()`, the engine computes a weighted combination of posterior draws across multiple metrics. The score is computed for each draw separately, then summarised. This means the uncertainty in each component metric propagates into the composite score rather than being hidden.

The composite score also supports a `deterioration_weights` term and a `guardrail_penalty` that reduces the score for guardrail failures. These are configured through the `config` dict.

## Decision States

The engine classifies each experiment into one of five states, and maps that state to a recommendation.

### strong win

**Conditions**: primary signal is strong, risk is low, effect is practically significant, all guardrails pass.

Strong signal means `P(best) >= prob_best_strong` (default 0.95) AND `expected_loss <= expected_loss_max` (default 0.01) AND `P(practical) >= rope_practical_min` (default 0.80).

Risk is low when `expected_loss <= expected_loss_max` and `CVaR / expected_loss <= cvar_ratio_max`.

**Recommendation**: ship variant.

### weak win

**Conditions**: primary signal is moderate, risk is low, effect is practically significant, all guardrails pass.

Moderate signal means `P(best) >= prob_best_moderate` (default 0.80) but the full strong criteria are not met.

**Recommendation**: consider shipping.

This verdict often means the experiment needs more data to reach the strong threshold, or the effect is real but the practical significance is borderline. Inspect the HDI and ROPE breakdown in `result.summary()` to understand which condition is not yet satisfied.

### high risk

**Conditions**: risk is high — `expected_loss > expected_loss_max * 5`.

The factor-of-5 buffer exists because expected loss in the moderate range is often a sign of genuine uncertainty rather than genuine risk. Only sustained high loss across the full distribution triggers this state.

**Recommendation**: do not ship.

### guardrail conflicts

**Conditions**: the primary signal is strong (the evidence would otherwise support shipping), but at least one guardrail fails.

The engine surfaces the conflict rather than hiding it or resolving it with an arbitrary formula. The conflict message names the specific guardrail and the P(degraded) value so you know exactly what the tradeoff is.

**Recommendation**: review required.

This is the correct outcome when a feature improves revenue but degrades latency. The framework has done its job by surfacing the conflict. The next step is a business decision: is the revenue gain worth the latency cost for your users?

### inconclusive

**Conditions**: none of the above states apply. The experiment does not yet have enough signal to make a call in either direction.

**Recommendation**: continue experiment.

If the experiment has run for a reasonable time and is still inconclusive, check whether the effect size is genuinely smaller than `min_effect` (which would mean stopping the experiment and recording no difference), or whether the experiment is underpowered for the effect size you care about.

## Primary Strength and Risk Classification Details

The primary signal has three levels — `"strong"`, `"moderate"`, `"weak"` — computed before the state determination. The risk level has three levels — `"low"`, `"medium"`, `"high"`. The confidence level, which feeds `result.confidence`, is computed independently from P(best) and the HDI width, and can be `"high"`, `"medium"`, or `"low"`.

These intermediate classifications appear in `result.summary()` under the DECISION section and in `result.reasons`. They are also accessible directly as attributes on the result object:

```python
result.primary_strength    # "strong" | "moderate" | "weak"
result.risk_level          # "low" | "medium" | "high"
result.practical_significance  # "yes" | "uncertain" | "no"
result.confidence          # "high" | "medium" | "low"
```

## Notes and Warnings

Both the model layer and the engine layer emit structured notes that surface in `result.notes` and are printed at the bottom of `result.summary()` under FLAGGED ISSUES.

Common notes:

- `[MCMC] ...` — convergence issue or unexpected posterior shape from the model layer.
- `Variant 'v' has extreme tail risk` — CVaR is more than 5x expected loss for that variant.
- `High severity guardrail violation on 'metric' for variant 'v'` — guardrail failed with severity classified as high.
- `[Segment 'name'] ...` — per-segment warning from the hierarchical model, forwarded to the appropriate segment result.

Notes do not block results. They are informational signals that something in the posterior or the decision logic deserves a closer look.
