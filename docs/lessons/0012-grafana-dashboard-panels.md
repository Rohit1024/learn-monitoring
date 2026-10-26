---
icon: lucide/notebook-pen
---

# 0012: Build Grafana Dashboards for Health, Traffic, Errors, and Latency

!!! note "Lesson outline"
    This page preserves the detailed session plan. Expand it into a full runnable lesson in a later session using the [authoring guide and shared lab contract](../lesson-authoring.md).

## Learning flow

``` mermaid
flowchart TD
    N0["Application health question"]
    N1["PromQL query"]
    N2["Choose visualization and units"]
    N3["Save dashboard panels"]
    N0 --> N1
    N1 --> N2
    N2 --> N3
```

## Session plan

**Lesson file:** `0012-grafana-dashboard-panels.md` · **Time:** 60–90 minutes

**Prerequisites:** 0010 and 0011.

**Outcome:** Save a dashboard answering health, throughput, error, latency, and concurrency questions.

**Teach:** Time Series, Stat, Gauge, Bar Gauge, Table, optional histogram Heatmap; instant/range queries, legends, reductions, units, thresholds, descriptions.

**Guided lab:**

1. Build health Stat, RPS Time Series, error-percentage Time Series, p95 Time Series, in-progress Gauge, and route traffic comparison.
2. Set seconds, requests/second, and percentage units matching query scales; explain Stat reduction semantics.
3. Exercise normal/slow/error traffic; inspect query results before styling. Export JSON and document import plus data source UID.

**Verification evidence:** Dashboard JSON and a description of each panel question, query, unit, and visualization choice.

**Failure / independent challenge:** Find and fix a 100× percentage-formatting error; add one independent panel.

**Active recall:** Why are cumulative totals a poor RPS panel? How do Last and Mean differ? Does a colored threshold create an alert rule?

**Official sources:** [Visualizations](https://grafana.com/docs/grafana/latest/visualizations/panels-visualizations/visualizations/), [Query editor](https://grafana.com/docs/grafana/latest/datasources/prometheus/query-editor/).

``` text
$teach Expand lesson 0012 using the outline in docs/lessons/0012-grafana-dashboard-panels.md and docs/lesson-authoring.md.
```

| Previous Lesson | All Lessons | Next Lesson |
| :--- | :---: | ---: |
| [0011: Connect Grafana to Prometheus and Provision the Data Source](0011-grafana-prometheus-data-source.md) | [Lesson index](index.md) | [0013: Create Dynamic Grafana Dashboards with Variables and Repeating Panels](0013-grafana-dashboard-variables.md) |
