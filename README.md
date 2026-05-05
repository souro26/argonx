# argonx

<!-- CI badge — live once GitHub Actions is connected -->
![CI](https://github.com/souro26/bayesian-a-b-testing/actions/workflows/ci.yml/badge.svg)
![License](https://img.shields.io/badge/license-MIT-blue.svg)
![Python](https://img.shields.io/badge/python-3.10%2B-blue.svg)
<!-- PyPI badges — add after upload -->
<!-- ![PyPI](https://img.shields.io/pypi/v/argonx.svg) -->
<!-- ![Downloads](https://img.shields.io/pypi/dm/argonx.svg) -->

**argonx** is a decision-support system for A/B experiments. It handles Bayesian inference, multi-metric risk management, hierarchical segment-aware analysis, and sequential stopping — and surfaces a complete evidential picture so the right decision is obvious.

Most testing frameworks answer *"is there an effect?"* argonx answers *"what should you do about it, and how much do you lose if you're wrong?"*

---

## Install

```bash
pip install git+https://github.com/souro26/bayesian-a-b-testing.git
```

```bash
# or, for local development
git clone https://github.com/souro26/bayesian-a-b-testing.git
cd bayesian-a-b-testing
pip install -e .
```

---

## Quick Example

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

For ratio metrics, pass a callable directly — no class system needed:

```python
experiment = Experiment(
    data=df,
    variant_col='variant',
    primary_metric=lambda df: df['clicks'] / df['impressions'],
    model='lognormal',
    control='control',
)
```

For segment-aware hierarchical inference, add one argument:

```python
experiment = Experiment(
    data=df,
    variant_col='variant',
    segment_col='device_type',          # triggers hierarchical model automatically
    primary_metric='revenue',
    model='lognormal',
    control='control',
)

result = experiment.run()
result.summary()           # aggregate, population-level
result.segment_summary()   # per-segment decisions + cross-segment conflict detection
```

---

## What It Computes

A p-value tells you the probability of seeing data this extreme if the null is true. It does not tell you what to do. argonx computes the quantities that actually drive decisions:

| Metric | What it answers |
|---|---|
| **P(variant is best)** | Which variant has the highest posterior probability of being the true winner — computed via simultaneous argmax across all N variants, not pairwise comparison |
| **Expected loss** | How much you lose on average if you ship the wrong variant — integrated over the full posterior, not a point estimate |
| **CVaR** | Expected loss in the worst-case tail — catches cases where the average loss looks fine but catastrophic outcomes are possible |
| **ROPE** | Is the effect large enough to matter in practice? An effect can be statistically real and business-irrelevant. ROPE separates these |
| **HDI** | The actual probability interval — not a frequentist confidence interval. The lift is inside this range with 95% posterior probability |
| **Joint probability** | P(all business conditions satisfied simultaneously) — not per-metric checks that miss correlations |
| **Composite score** | Weighted multi-metric business impact, computed draw-by-draw from posteriors |
| **Guardrail conflict** | When the primary metric improves and a guardrail degrades, the framework surfaces the conflict clearly rather than resolving it arbitrarily |
| **Sequential stopping** | Evidence-based stopping signal. Stop when expected loss drops below threshold — not when a fixed sample size is hit |

### Why not just use a t-test?

A t-test answers one question: is the observed difference unlikely under the null? It cannot tell you:

- How much you lose if you ship and you're wrong
- Whether the effect is large enough to change user behaviour
- What to do when conversion improves but latency degrades
- Whether it's safe to stop the experiment early
- How thin-segment estimates should borrow strength from larger segments

argonx answers all of these. The decision engine is the project — the models are plumbing.

### Why not just use PyMC directly?

PyMC gives you posteriors and stops there. It has no concept of which variant to ship, what your business risk tolerance is, or whether your full policy is satisfied simultaneously. argonx is a genuine layer on top of PyMC — not a wrapper, not a replacement.

---

## Models

| Model | Use case | Data type |
|---|---|---|
| `binary` | Conversion rate, click-through, churn | 0/1 outcomes |
| `lognormal` | Revenue, order value, session duration | Right-skewed positive continuous |
| `gaussian` | Latency, load time, scores | Symmetric continuous |
| `studentt` | Same as gaussian but robust to outliers | Symmetric continuous with heavy tails |
| `poisson` | Events per user, purchases per session | Count data |

Every model has a flat and hierarchical variant. Flat is selected by default. Hierarchical is selected automatically when `segment_col` is provided — no additional configuration required.

Guardrail metrics can use a different model than the primary metric:

```python
experiment = Experiment(
    ...
    model='binary',                              # primary: conversion rate
    guardrail_models={'page_load_ms': 'lognormal'},  # guardrail: load time
)
```

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
Effect is OUTSIDE ROPE — practically meaningful.
P(practical effect): 0.941

GUARDRAILS
----------------------------------------
  page_load_ms              [FAIL]  variant=variant_b  P(degraded)=0.912  threshold=0.100

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

## Sequential Stopping

```python
from argonx.sequential import StoppingChecker

checker = StoppingChecker(
    loss_threshold=0.01,
    prob_best_min=0.95,
    min_sample_size=1000,
)

# called at each checkpoint as data accumulates
status = checker.update(
    samples=result.samples,
    variant_names=['control', 'variant_b'],
    control='control',
    n_users_per_variant=n_counts,
)

print(status.safe_to_stop)
print(status.users_needed)   # approximate additional users if not safe to stop

checker.plot_trajectory()    # evidence accumulation over time
```

Bayesian sequential testing is valid at any checkpoint. Frequentist peeking inflates false positive rates — Bayesian expected-loss stopping does not. argonx tells you when evidence is strong enough, not when a predetermined sample size is reached.

---

## Examples

Five real-world worked examples in [`examples/`](examples/):

| Notebook | Scenario | Key feature |
|---|---|---|
| `01_ecommerce_checkout.ipynb` | Checkout redesign | Guardrail conflict: conversion vs. load time |
| `02_saas_revenue_sequential.ipynb` | SaaS pricing page | Sequential stopping fires early |
| `03_clinical_trial.ipynb` | Drug dosage protocol | StudentT vs Gaussian: outlier robustness |
| `04_gaming_matchmaking.ipynb` | Matchmaking algorithm | 3-way multivariant, simultaneous argmax |
| `05_mobile_personalisation.ipynb` | Fintech personalisation | Hierarchical: iOS wins, Android neutral, thin tablet segment |

---

## Running Tests

```bash
# Fast — unit tests only, no MCMC (~60 seconds)
pytest tests/unit/

# Statistical property verification — no MCMC
pytest tests/math/

# Full suite including MCMC integration tests (slow)
pytest tests/
```

The test suite has three tiers matching the CI pipeline. Unit tests run on every push. Math tests run on every PR. Integration tests run on merge to main.

---

## Contributing

Bug reports and PRs are welcome. Before opening a PR:

- Run `pytest tests/unit/ tests/math/` and confirm everything passes
- For changes to the decision engine, add a test to `tests/math/test_decision_sims.py` that verifies the statistical property you're changing
- For new model variants, add corresponding tests to `tests/integration/test_models.py`

Open an issue first for anything beyond bug fixes — architectural changes to the decision engine or new model types are worth discussing before implementation.

---

## License

MIT
