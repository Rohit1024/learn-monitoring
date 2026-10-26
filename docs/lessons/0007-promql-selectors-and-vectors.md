---
icon: lucide/notebook-pen
---

# 0007: Select and Filter Time Series with PromQL Vectors and Time Windows

!!! note "Lesson outline"
    This page preserves the detailed session plan. Expand it into a full runnable lesson in a later session using the [authoring guide and shared lab contract](../lesson-authoring.md).

## Learning flow

``` mermaid
flowchart TD
    N0["Metric name and label matchers"]
    N1["Select samples using time windows"]
    N2["Instant or range vectors"]
    N3["Evaluate functions and expressions"]
    N0 --> N1
    N1 --> N2
    N2 --> N3
```

## Session plan

**Lesson file:** `0007-promql-selectors-and-vectors.md` · **Time:** 60–90 minutes

**Prerequisites:** 0006.

**Outcome:** Select the intended series and explain query types and time windows.

**Teach:** Instant/range vectors, scalars, strings; `=`, `!=`, `=~`, `!~`; anchored regexes, absent labels, `[5m]`, `offset`, introductory `@`, lookback, and staleness.

**Guided lab:**

1. Filter by method, route template, and status; predict remaining series and labels.
2. Inspect a bare `[5m]` range vector in Table, then use a suitable function for a graphable result.
3. Accumulate history and compare current selection with `offset 5m`; inspect a stopped target.

**Verification evidence:** Ten exercises with predicted and observed result types, labels, and time semantics.

**Failure / independent challenge:** Repair a regex selecting unintended routes and a selector unintentionally matching absent labels.

**Active recall:** Is an instant-vector sample necessarily scraped exactly at evaluation time? Does `[5m]` mean a five-minute graph? Why is a bare range vector invalid as a Graph/range-query root?

**Official sources:** [PromQL basics](https://prometheus.io/docs/prometheus/latest/querying/basics/).

``` text
$teach Expand lesson 0007 using the outline in docs/lessons/0007-promql-selectors-and-vectors.md and docs/lesson-authoring.md.
```

| Previous Lesson | All Lessons | Next Lesson |
| :--- | :---: | ---: |
| [0006: Explore Metrics in the Prometheus UI and HTTP Query API](0006-prometheus-ui-and-query-api.md) | [Lesson index](index.md) | [0008: Calculate Request Rates and Error Ratios with Counter Reset Handling](0008-promql-rates-and-counter-resets.md) |
