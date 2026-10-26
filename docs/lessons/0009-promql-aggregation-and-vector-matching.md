---
icon: lucide/notebook-pen
---

# 0009: Aggregate Metrics and Match Vectors Across Routes and Instances

!!! note "Lesson outline"
    This page preserves the detailed session plan. Expand it into a full runnable lesson in a later session using the [authoring guide and shared lab contract](../lesson-authoring.md).

## Learning flow

``` mermaid
flowchart TD
    N0["Labeled metric series"]
    N1["Calculate per-series rates"]
    N2["Aggregate selected dimensions"]
    N3["Match vectors for arithmetic"]
    N0 --> N1
    N1 --> N2
    N2 --> N3
```

## Session plan

**Lesson file:** `0009-promql-aggregation-and-vector-matching.md` · **Time:** 60–90 minutes

**Prerequisites:** 0008.

**Outcome:** Compare routes/instances and combine vectors without silently losing series.

**Teach:** `sum`, `avg`, `min`, `max`, `count`, `topk`; `by`/`without`; arithmetic/comparisons; default matching, `on`, `ignoring`, and a bounded `group_left` example.

**Guided lab:**

1. Add `app2:8000` to the same job with a distinct `instance`; generate unequal traffic.
2. Group RPS by route, method, and instance; align numerator/denominator labels for route error ratios.
3. Compare ratio-of-sums with unweighted average-of-ratios; break matching and repair it. Explain rejected many-to-many arithmetic.

``` promql
sum by (route) (rate(app_http_requests_total{job="demo-api"}[5m]))
```

**Verification evidence:** A worksheet recording input labels, grouping, output labels, units, and why heavy-traffic instances need appropriate weight.

**Failure / independent challenge:** Diagnose an empty vector arithmetic result and a many-to-many matching error.

**Active recall:** What does `by(route)` discard? How does `without` change retained dimensions? Why rate before summing counters?

**Official sources:** [PromQL operators](https://prometheus.io/docs/prometheus/latest/querying/operators/).

``` text
$teach Expand lesson 0009 using the outline in docs/lessons/0009-promql-aggregation-and-vector-matching.md and docs/lesson-authoring.md.
```

| Previous Lesson | All Lessons | Next Lesson |
| :--- | :---: | ---: |
| [0008: Calculate Request Rates and Error Ratios with Counter Reset Handling](0008-promql-rates-and-counter-resets.md) | [Lesson index](index.md) | [0010: Query Latency Percentiles and Thresholds from Histograms](0010-promql-histograms-and-latency.md) |
