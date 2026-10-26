---
icon: lucide/notebook-pen
---

# 0014: Transform Query Results into Grafana Summary Tables

!!! note "Lesson outline"
    This page preserves the detailed session plan. Expand it into a full runnable lesson in a later session using the [authoring guide and shared lab contract](../lesson-authoring.md).

## Learning flow

``` mermaid
flowchart TD
    N0["Query result data frames"]
    N1["Join on route"]
    N2["Organize and format fields"]
    N3["Summary table"]
    N0 --> N1
    N1 --> N2
    N2 --> N3
```

## Session plan

**Lesson file:** `0014-grafana-transformations-and-tables.md` · **Time:** 60–90 minutes

**Prerequisites:** 0013.

**Outcome:** Build a route summary table traceable to original query results.

**Teach:** Data frames, transformation order, joins, field calculations, organization, sorting, overrides. Prefer PromQL for reset-aware rates and service ratios; use transformations to shape presentation.

**Guided lab:**

1. Run instant queries for RPS, error percentage, and p95 grouped by identical `route` labels.
2. Inspect frames, join by route, organize/rename fields, set units/overrides, and sort.
3. Compare cells to original queries under variable filters; choose null and missing-row handling explicitly.

**Verification evidence:** Exported table panel plus pipeline documentation including join keys and null behavior.

**Failure / independent challenge:** Join inconsistent label sets, investigate duplicates/missing rows, and repair the pipeline.

**Active recall:** How do inner and outer joins differ? Why does transformation order matter? Why should alerts not depend on panel transformations?

**Official sources:** [Transform data](https://grafana.com/docs/grafana/latest/visualizations/panels-visualizations/query-transform-data/transform-data/).

``` text
$teach Expand lesson 0014 using the outline in docs/lessons/0014-grafana-transformations-and-tables.md and docs/lesson-authoring.md.
```

| Previous Lesson | All Lessons | Next Lesson |
| :--- | :---: | ---: |
| [0013: Create Dynamic Grafana Dashboards with Variables and Repeating Panels](0013-grafana-dashboard-variables.md) | [Lesson index](index.md) | [0015: Build an End-to-End Golden Signals Monitoring Stack](0015-golden-signals-monitoring-project.md) |
