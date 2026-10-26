---
icon: lucide/notebook-pen
---

# 0010: Query Latency Percentiles and Thresholds from Histograms

!!! note "Lesson outline"
    This page preserves the detailed session plan. Expand it into a full runnable lesson in a later session using the [authoring guide and shared lab contract](../lesson-authoring.md).

## Learning flow

``` mermaid
flowchart TD
    N0["Cumulative latency buckets"]
    N1["Calculate bucket rates"]
    N2["Aggregate preserving le and route"]
    N3["Estimate latency percentiles"]
    N0 --> N1
    N1 --> N2
    N2 --> N3
```

## Session plan

**Lesson file:** `0010-promql-histograms-and-latency.md` · **Time:** 60–90 minutes

**Prerequisites:** 0009.

**Outcome:** Query mean latency, p50/p95/p99, and the fraction of requests below a latency threshold.

**Teach:** Cumulative classic buckets, `le`, boundary choice, quantile interpolation, and cross-instance aggregation. Quantiles are estimates; preserve `le` when summing classic buckets. Compare native-histogram query shape.

**Guided lab:**

1. Predict bucket changes for known observations and compare `_count`/`_sum`.
2. Generate mixed slow/normal traffic; graph mean and quantiles. Use the exact 0.3-second bucket for a 300 ms threshold fraction.
3. Combine instance buckets before calculating quantiles, and contrast with invalid averaging of instance p95 values.

``` promql
histogram_quantile(0.95,
  sum by (le, route) (
    rate(app_http_request_duration_seconds_bucket{job="demo-api"}[5m])
  )
)
```

**Verification evidence:** Latency queries documenting seconds, grouping labels, bucket boundaries, approximation limits, and no-observation behavior.

**Failure / independent challenge:** Remove `le` from classic bucket aggregation, diagnose the result, then restore it.

**Active recall:** Why can coarse buckets obscure a regression? How does mean differ from p95? Why does this bucket query not work on Python Summary?

**Official sources:** [Histogram functions](https://prometheus.io/docs/prometheus/latest/querying/functions/), [Python Histogram](https://prometheus.github.io/client_python/instrumenting/histogram/).

``` text
$teach Expand lesson 0010 using the outline in docs/lessons/0010-promql-histograms-and-latency.md and docs/lesson-authoring.md.
```

| Previous Lesson | All Lessons | Next Lesson |
| :--- | :---: | ---: |
| [0009: Aggregate Metrics and Match Vectors Across Routes and Instances](0009-promql-aggregation-and-vector-matching.md) | [Lesson index](index.md) | [0011: Connect Grafana to Prometheus and Provision the Data Source](0011-grafana-prometheus-data-source.md) |
