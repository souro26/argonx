# Models

argonx ships five generative models, each available in a flat variant (one model per session, all variants independent) and a hierarchical variant (partial pooling across segments via a shared hyperprior). You select the flat or hierarchical variant by setting or omitting `segment_col` on `Experiment` — the switch happens automatically.

## Choosing a Model

The short version:

| Data | Model |
|---|---|
| Click, convert, churn (0/1 per user) | `binary` |
| Revenue, order value, session duration (positive, right-skewed) | `lognormal` |
| Latency, score delta, NPS (roughly symmetric) | `gaussian` |
| Same as gaussian but with visible outliers | `studentt` |
| Events per user, purchases per session (integer counts) | `poisson` |

The longer version follows below.

## binary

**Use when**: each observation is a binary outcome — a user either converted or did not, clicked or did not, churned or did not.

**Model**: Beta-Bernoulli conjugate update. The prior on conversion rate `p` is `Beta(1, 1)` (uniform). After observing `k` successes from `n` trials, the posterior is `Beta(k+1, n-k+1)`. This is exact — no MCMC, no approximation. `n_draws` controls how many samples are drawn from this exact posterior.

**Returns**: posterior samples of the conversion rate `p ∈ (0, 1)`.

**When not to use it**: if your data is not truly binary (e.g., revenue binned into bought/did-not-buy), prefer `lognormal` for the continuous values. The binary model throws away the magnitude of continuous outcomes.

## lognormal

**Use when**: the metric is positive and right-skewed. Revenue per user, average order value, session duration, and time-on-page all fit here. The defining characteristic is that the log of the values is roughly symmetric.

**Model**: LogNormal likelihood with a Normal prior on the log-mean `μ`. The posterior is sampled via MCMC. The returned values are posterior samples of the expected value `E[X] = exp(μ + σ²/2)` rather than samples of `μ` directly, so the decision engine operates on the scale that actually matters.

**Returns**: posterior samples of `E[X]`, one per draw per variant.

**Note**: extreme parameter values can cause `exp()` to overflow. The engine checks for NaN/Inf in samples and raises `ValueError` if found. If this happens, check whether your data contains very large outliers or whether the variant has very few observations.

## gaussian

**Use when**: the metric is continuous and roughly symmetric — latency in milliseconds, normalised score deltas, or any metric where the distribution does not have a heavy positive tail.

**Model**: Normal likelihood with a Normal prior on the mean and a HalfNormal prior on the standard deviation.

**Returns**: posterior samples of the mean.

**Flat vs studentt**: if your data has occasional extreme values (a single user with 10x the typical latency), `studentt` is strictly more robust because the heavier tails of the Student-T likelihood downweight those outliers automatically. On clean data, `gaussian` and `studentt` converge to the same posterior.

## studentt

**Use when**: the same conditions as `gaussian`, but the data has visible outliers that you do not want to drop.

**Model**: Student-T likelihood with Normal and HalfNormal priors on location and scale, and a `Gamma(2, 0.1)` prior on the degrees-of-freedom parameter `ν`. The `ν` parameter is learned from the data — low values indicate heavy tails, high values converge toward Gaussian.

**Returns**: posterior samples of the location parameter (the robust mean estimate).

**Practical note**: on genuinely Gaussian data, `studentt` adds MCMC overhead without meaningful accuracy gain. On data with outliers, it is often the right call.

## poisson

**Use when**: the metric counts discrete events — errors per session, purchases per user, API calls per minute, support tickets opened per account.

**Model**: Gamma-Poisson conjugate. The prior on the rate `λ` is `Gamma(1, 1)`. After observing total events `k` from `n` observations, the posterior is `Gamma(k+1, n+1)`. Like binary, this is exact.

**Returns**: posterior samples of the event rate `λ` (expected events per observation).

**When not to use it**: if the count distribution has excess zeros or if variance is much higher than the mean (overdispersion), consider whether the data might fit better as a continuous positive metric modelled with `lognormal`.

## Hierarchical Models

When `segment_col` is set on `Experiment`, argonx automatically selects the hierarchical variant of the chosen model. You do not instantiate these directly.

Hierarchical models accept nested data as `dict[segment, dict[variant, np.ndarray]]` and produce two sets of posterior samples: population-level samples (via `sample_posterior()`, used for the aggregate decision) and per-segment samples (via `sample_posterior_by_segment()`, used for `result.segment_summary()`).

The key property of partial pooling: a segment with 50 observations does not get an independent posterior estimated from 50 points. Instead, its posterior is regularised toward the population mean in proportion to how much the segment data supports diverging from it. A segment with 5000 observations can diverge freely — the hyperprior has little influence. A segment with 50 observations is pulled toward the population estimate. This is almost always the right behaviour.

### Thin-segment warnings

The engine emits a warning when a segment's posterior is dominated by the hyperprior rather than its own data. The message appears in `result.notes` and in `result.segment_summary()`. The warning does not block the result — the inference is valid — but it tells you that the segment's individual estimate is not carrying much independent information.

## Prior Overrides

For hierarchical models, you can override the default hyperpriors through the `priors` argument on `Experiment`. The accepted keys depend on the model.

**HierarchicalBinaryModel**

```python
priors={
    "alpha_mean": 2.0,   # prior mean of the Beta alpha parameter
    "beta_mean": 2.0,    # prior mean of the Beta beta parameter
}
```

**HierarchicalLogNormalModel**

```python
priors={
    "mu_mean": 0.0,      # prior mean of log-mean hyperprior
    "mu_sigma": 1.0,     # prior std of log-mean hyperprior
}
```

**HierarchicalGaussianModel** / **HierarchicalStudentTModel**

```python
priors={
    "mu_mean": 0.0,
    "mu_sigma": 10.0,
    "sigma_beta": 5.0,   # scale parameter for the HalfNormal prior on std
}
```

**HierarchicalPoissonModel**

```python
priors={
    "alpha": 2.0,        # Gamma hyperprior shape
    "beta": 1.0,         # Gamma hyperprior rate
}
```

Prior overrides only affect hierarchical models. They have no effect when `segment_col` is not set, because flat models use fixed conjugate or weakly informative priors that are not exposed.

## MCMC Convergence

Flat conjugate models (binary, poisson) do not use MCMC. Their posteriors are exact regardless of `n_draws`.

MCMC models (lognormal, gaussian, studentt, and all hierarchical variants) use PyMC under the hood. The engine does not expose chain diagnostics (R-hat, ESS) through the main API — those are accessible through PyMC's trace directly if you need them. The engine does emit a warning if the posterior samples contain NaN or Inf values, which is the most common sign of a convergence failure worth investigating.

For hierarchical models, MCMC warnings are routed into `result.notes` with a `[MCMC]` prefix, and per-segment warnings appear with a `[Segment 'name']` prefix.
