---
icon: lucide/notebook-pen
---

# 0008: Calculate Request Rates and Error Ratios with Counter Reset Handling

!!! note "Lesson outline"
    This page preserves the detailed session plan. Expand it into a full runnable lesson in a later session using the [authoring guide and shared lab contract](../lesson-authoring.md).

## Learning flow

``` mermaid
flowchart TD
    N0["Request counters per series"]
    N1["Reset-aware rate calculation"]
    N2["Aggregate request and error rates"]
    N3["Throughput and error ratio"]
    N0 --> N1
    N1 --> N2
    N2 --> N3
```

## Session plan

**Lesson file:** `0008-promql-rates-and-counter-resets.md` · **Time:** 60–90 minutes

**Prerequisites:** 0007.

**Outcome:** Answer “How busy is the API?” and “What fraction of requests fail?” from counters.

**Teach:** `rate`, `irate`, `increase`, `resets`, per-second units, window selection, extrapolation, and reset correction. `increase` can be fractional; `irate` uses the last two samples.

**Guided lab:**

1. Generate steady traffic, a burst, and a known success/error mix; compare raw counters and one-/five-minute rates.
2. Compute total RPS and error percentage with the baseline expressions below; introduce aggregation briefly before lesson 0009.
3. Restart the app; compare reset-aware rates with naive subtraction. Stop traffic and distinguish zero throughput from an undefined zero-denominator ratio.

``` promql
sum(rate(app_http_requests_total{job="demo-api"}[5m]))

100 * sum(rate(app_http_requests_total{job="demo-api",status=~"5.."}[5m]))
  / sum(rate(app_http_requests_total{job="demo-api"}[5m]))
```

**Verification evidence:** Queries with correct units, a reset timeline, and separate interpretations for absent errors, insufficient samples, and no traffic.

**Failure / independent challenge:** Apply `rate` to a gauge and explain the misleading result. Do not blanket-convert empty/undefined results to zero.

**Active recall:** Why rate each series before summing? Why may `increase` differ from exact generated counts? How does window length affect smoothing?

**Official sources:** [Counter functions](https://prometheus.io/docs/prometheus/latest/querying/functions/).

``` text
$teach Expand lesson 0008 using the outline in docs/lessons/0008-promql-rates-and-counter-resets.md and docs/lesson-authoring.md.
```

| Previous Lesson | All Lessons | Next Lesson |
| :--- | :---: | ---: |
| [0007: Select and Filter Time Series with PromQL Vectors and Time Windows](0007-promql-selectors-and-vectors.md) | [Lesson index](index.md) | [0009: Aggregate Metrics and Match Vectors Across Routes and Instances](0009-promql-aggregation-and-vector-matching.md) |
