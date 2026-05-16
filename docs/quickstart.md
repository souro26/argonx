# Quickstart

## Installation

```bash
pip install argonx
```

For development, clone and install in editable mode:

```bash
git clone https://github.com/souro26/argonx.git
cd argonx
pip install -e ".[dev]"
```

Runtime dependencies are numpy, pandas, scipy, matplotlib, pymc, and arviz. The `[dev]` extra adds pytest, ruff, pre-commit, and notebook support.

## Your First Experiment

The entry point for everything is `Experiment`. Give it your data, tell it which column identifies the variants, which column is the metric, and which model fits your data type.

```python
import pandas as pd
from argonx import Experiment

df = pd.read_csv("experiment_data.csv")

experiment = Experiment(
    data=df,
    variant_col="variant",
    primary_metric="revenue",
    model="lognormal",
    control="control",
)

result = experiment.run()
result.summary()
```

`result.summary()` prints the full decision output — lift, risk, ROPE, guardrails, and a plain-English recommendation. See [results.md](results.md) for a breakdown of every field.

## Guardrails

A guardrail is a secondary metric that must not degrade for the decision to be clean. If the primary metric improves but a guardrail degrades, the framework surfaces a conflict and stops there rather than hiding it in an average.

```python
experiment = Experiment(
    data=df,
    variant_col="variant",
    primary_metric="revenue",
    model="lognormal",
    guardrails=["page_load_ms", "error_rate"],
    lower_is_better={"page_load_ms": True, "error_rate": True},
    guardrail_models={
        "page_load_ms": "gaussian",
        "error_rate": "binary",
    },
    control="control",
)
```

`lower_is_better` tells the engine which direction counts as degradation. `guardrail_models` lets each guardrail use a different model than the primary — latency fits a Gaussian, error rate fits a Binary. If a guardrail is not listed in `guardrail_models`, the primary model is used as fallback.

## Ratio Metrics

Pass a callable instead of a column name for derived metrics. The callable receives the full dataframe for a given variant group.

```python
experiment = Experiment(
    data=df,
    variant_col="variant",
    primary_metric=lambda df: df["clicks"] / df["impressions"],
    model="lognormal",
    control="control",
)
```

## Segment-Aware Inference

Adding `segment_col` switches the experiment to hierarchical mode. Each segment gets its own posterior estimate, but a shared hyperprior pools information across segments — thin segments borrow strength from larger ones without collapsing to the population mean.

```python
experiment = Experiment(
    data=df,
    variant_col="variant",
    segment_col="device_type",
    primary_metric="revenue",
    model="lognormal",
    control="control",
)

result = experiment.run()
result.summary()          # population-level aggregate
result.segment_summary()  # per-segment decisions with cross-segment conflict detection
```

`segment_summary()` also detects shipping conflicts — situations where one segment supports shipping and another does not. It tells you which segments drive the conflict rather than averaging them away.

## Tuning the Decision Thresholds

The engine ships with defaults that work well for most product experiments. You can override any of them through the `config` argument on `run()`.

```python
result = experiment.run(
    min_effect=0.02,       # minimum lift that counts as practically meaningful
    config={
        "prob_best_strong": 0.97,     # raise the bar for a "strong win" verdict
        "expected_loss_max": 0.005,   # tighter loss tolerance
    },
)
```

The full list of config keys, their types, and their defaults is in [experiment.md](experiment.md).

## Composite Scoring

When you have multiple metrics that all matter and want to rank variants by a weighted combination, use `composite_weights`. The score is computed draw-by-draw from posteriors, not from means, so uncertainty propagates correctly.

```python
result = experiment.run(
    composite_weights={
        "revenue": 0.6,
        "page_load_ms": 0.4,
    },
)
```

## Sequential Stopping

If you are running a live experiment and checking it periodically, use `StoppingChecker`. It maintains a trajectory of checkpoints and tells you whether evidence is strong enough to stop, and if not, estimates how many more users you need.

```python
from argonx.sequential import StoppingChecker

checker = StoppingChecker(
    loss_threshold=0.01,
    prob_best_min=0.95,
    min_sample_size=1000,
)

# call this at each checkpoint as new data arrives
status = checker.update(
    samples=result.samples,
    variant_names=["control", "variant_b"],
    control="control",
    n_users_per_variant={"control": 1200, "variant_b": 1195},
)

print(status.safe_to_stop)
print(status.recommendation)

checker.plot_trajectory()
```

See [sequential.md](sequential.md) for the full parameter reference and an explanation of the gate sequence.

## Reading Plots

```python
result.plot(metric_name="revenue")
```

This renders a five-panel dashboard: posterior distributions, lift with HDI, P(best) and expected loss per variant, ROPE analysis, and guardrail probabilities. For hierarchical experiments, `result.plot_segments()` produces a grid of posterior plots by segment.

## Exporting Results

```python
df = result.to_dataframe()   # per-variant metrics as a DataFrame
d = result.to_dict()         # fully serialisable dict, safe for JSON
```

Both methods include joint probability and composite score columns when those were computed.
