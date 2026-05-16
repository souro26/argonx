# argonx

![CI](https://github.com/souro26/bayesian-a-b-testing/actions/workflows/ci.yml/badge.svg)
![PyPI](https://img.shields.io/pypi/v/argonx.svg)
![Downloads](https://static.pepy.tech/personalized-badge/argonx?period=total&units=INTERNATIONAL_SYSTEM&left_color=BLACK&right_color=GREEN&left_text=downloads)
![Python](https://img.shields.io/badge/python-3.10%2B-blue.svg)
![License](https://img.shields.io/badge/license-MIT-blue.svg)

**argonx** is a Bayesian decision engine for A/B experiments.

It handles inference, multi-metric risk management, hierarchical segment-aware analysis, and sequential stopping. Feed it your data, tell it what matters, and it surfaces everything you need to make the right call.

---

## Install

```bash
pip install argonx
```

```bash
# development install
git clone https://github.com/souro26/argonx.git
cd argonx
pip install -e .
```

---

## Quick Start

```python
from argonx import Experiment

experiment = Experiment(
    data=df,
    variant_col='variant',
    primary_metric='revenue',
    guardrails=['page_load_ms'],
    lower_is_better={'page_load_ms': True},
    model='lognormal',
    guardrail_models={'page_load_ms': 'gaussian'},
    control='control',
)

result = experiment.run()
result.summary()
result.plot()
```

Ratio metrics via callable, no extra classes needed:

```python
experiment = Experiment(
    data=df,
    variant_col='variant',
    primary_metric=lambda df: df['clicks'] / df['impressions'],
    model='lognormal',
    control='control',
)
```

Segment-aware hierarchical inference, one extra argument:

```python
experiment = Experiment(
    data=df,
    variant_col='variant',
    segment_col='device_type',
    primary_metric='revenue',
    model='lognormal',
    control='control',
)

result = experiment.run()
result.summary()          # aggregate, population-level
result.segment_summary()  # per-segment decisions, cross-segment conflict detection
```

---

## What It Computes

Most testing frameworks answer *is there an effect?* argonx answers what you should do about it, and how much you lose if you get it wrong.

A p-value tells you whether the observed difference is unlikely under the null. It does not tell you which variant to ship. argonx computes the quantities that actually drive that decision:

| Metric | What it answers |
|---|---|
| **P(variant is best)** | Posterior probability of being the true winner, computed via simultaneous argmax across all N variants. Not pairwise. |
| **Expected loss** | Average loss if you ship the wrong variant, integrated over the full posterior. Not a point estimate. |
| **CVaR** | Expected loss in the worst-case tail. Catches cases where average loss looks fine but tail outcomes are catastrophic. |
| **ROPE** | Is the effect large enough to matter in practice? A statistically real effect can still be business-irrelevant. ROPE separates these. |
| **HDI** | The actual posterior probability interval. The lift falls inside this range with 95% posterior probability. |
| **Joint probability** | P(all business conditions satisfied simultaneously), not independent per-metric checks that miss correlations. |
| **Composite score** | Weighted multi-metric business impact, computed draw-by-draw from posteriors, not from means. |
| **Guardrail conflict** | When the primary metric improves and a guardrail degrades, the framework surfaces the conflict and stops there. No arbitrary resolution. |
| **Sequential stopping** | Stop when expected loss drops below your threshold, not when a fixed sample size is reached. |

---

## What `result.summary()` Looks Like

```
============================================================
EXPERIMENT RESULTS
============================================================

PRIMARY METRIC
----------------------------------------
Best Variant: variant_b
Expected lift:    +4.3% (95% HDI: +1.0% to +7.0%)
P(best) across all variants: 0.971

RISK
----------------------------------------
Expected loss if wrong:          0.0009
CVaR (95th percentile loss):     0.0021
Risk level:                      low

PRACTICAL SIGNIFICANCE (ROPE)
----------------------------------------
Effect is OUTSIDE ROPE -- practically meaningful.
P(practical effect): 0.941

GUARDRAILS
----------------------------------------
  page_load_ms    [FAIL]  variant=variant_b  P(degraded)=0.912  threshold=0.100

GUARDRAIL CONFLICTS DETECTED
----------------------------------------
Strong evidence for variant_b on primary metric.
Guardrail violation on page_load_ms with 91.2% probability.
Framework cannot resolve this tradeoff. Human review required.

============================================================
DECISION
----------------------------------------
State:          conflict
Recommendation: REVIEW REQUIRED
Confidence:     low

Reasoning:
  - P(best) exceeds strong threshold
  - Expected loss below configured maximum
  - Guardrail violation: page_load_ms cannot be resolved automatically
============================================================
```

The framework does not make the decision. It makes the right decision obvious.

---

## Models

| Model | Use case | Data type |
|---|---|---|
| `binary` | Conversion rate, click-through, churn | 0/1 outcomes |
| `lognormal` | Revenue, order value, session duration | Right-skewed positive continuous |
| `gaussian` | Latency, load time, scores | Symmetric continuous |
| `studentt` | Same as gaussian, robust to outliers | Symmetric continuous with heavy tails |
| `poisson` | Events per user, purchases per session | Count data |

Every model has a flat and hierarchical variant. Flat is the default. Hierarchical is selected automatically when `segment_col` is set. Partial pooling handles thin segments by borrowing strength from larger ones without collapsing differences that are real.

Guardrail metrics can use a different model than the primary:

```python
experiment = Experiment(
    ...
    model='binary',
    guardrail_models={'page_load_ms': 'lognormal'},
)
```

---

## Sequential Stopping

```python
from argonx.sequential import StoppingChecker

checker = StoppingChecker(
    loss_threshold=0.01,
    prob_best_min=0.95,
    min_sample_size=1000,
)

status = checker.update(
    samples=result.samples,
    variant_names=['control', 'variant_b'],
    control='control',
    n_users_per_variant=n_counts,
)

print(status.safe_to_stop)
print(status.users_needed)  # estimated additional users needed if not safe

checker.plot_trajectory()   # P(best) and expected loss over time
```

Frequentist peeking inflates false positive rates. Bayesian expected-loss stopping does not. argonx stops when evidence is strong enough, and tells you how far you are from that threshold when it is not.

---

## Examples

Five worked examples across different industries and model types in [`examples/`](examples/):

| Notebook | Scenario | Key feature |
|---|---|---|
| `01_ecommerce_checkout.ipynb` | Checkout redesign | Guardrail conflict: conversion up, load time up |
| `02_saas_revenue_sequential.ipynb` | SaaS pricing page | Sequential stopping fires at week 2 of 4 |
| `03_clinical_trial.ipynb` | Drug dosage protocol | StudentT vs Gaussian on data with outliers |
| `04_gaming_matchmaking.ipynb` | Matchmaking algorithm | 3-way experiment, simultaneous argmax |
| `05_mobile_personalisation.ipynb` | Fintech personalisation | Hierarchical: segment conflict, thin-segment pooling |

---

## Running Tests

```bash
# unit tests only, no MCMC, fast
pytest tests/unit/

# statistical property verification, no MCMC
pytest tests/math/

# full suite including MCMC integration tests
pytest tests/
```

Three tiers matching the CI pipeline. Unit tests on every push. Math tests on every PR. Integration tests on merge to main.

---

## Contributing

Open an issue before submitting anything beyond a bug fix. PRs are welcome.

Before opening a PR, run `pytest tests/unit/ tests/math/` and confirm everything passes. For decision engine changes, add a test to `tests/math/test_decision_sims.py` that verifies the statistical property. For new model variants, add tests to `tests/integration/test_models.py`.

---

## License

MIT
