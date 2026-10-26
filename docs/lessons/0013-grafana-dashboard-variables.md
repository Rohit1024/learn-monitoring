---
icon: lucide/notebook-pen
---

# 0013: Create Dynamic Grafana Dashboards with Variables and Repeating Panels

!!! note "Lesson outline"
    This page preserves the detailed session plan. Expand it into a full runnable lesson in a later session using the [authoring guide and shared lab contract](../lesson-authoring.md).

## Learning flow

``` mermaid
flowchart TD
    N0["Dashboard variable selections"]
    N1["Interpolate filters and rate windows"]
    N2["Run filtered PromQL queries"]
    N3["Render panels and repetitions"]
    N0 --> N1
    N1 --> N2
    N2 --> N3
```

## Session plan

**Lesson file:** `0013-grafana-dashboard-variables.md` · **Time:** 60–90 minutes

**Prerequisites:** 0009 and 0012.

**Outcome:** Reuse one dashboard across services, instances, and routes without editing panel queries.

**Teach:** Query/chained variables, multi-select, All, refresh policies, repeated panels/rows. Prefer Label values query type; legacy `label_values(...)` is Grafana syntax, not PromQL.

**Guided lab:**

1. Create `env`, `service`, `instance`, and `route` variables from existing labels; one `local` environment is sufficient initially.
2. Apply filters consistently; use regex matchers for multi-select. Test one instance, both, and All.
3. Replace fixed rate windows with `$__rate_interval`; inspect expansion while zooming. Contrast `$__interval`, query step, and `$__range`.

**Verification evidence:** Exported dashboard whose applicable panels respect filters, with bounded repetitions.

**Failure / independent challenge:** Break multi-select with `=` and repair it. Add a repeated per-instance panel.

**Active recall:** Why does Grafana variable syntax fail in Prometheus UI? Which setting supplies the scrape interval for `$__rate_interval`? Why should alerts avoid relying on dashboard variable state?

**Official sources:** [Template variables](https://grafana.com/docs/grafana/latest/datasources/prometheus/template-variables/).

``` text
$teach Expand lesson 0013 using the outline in docs/lessons/0013-grafana-dashboard-variables.md and docs/lesson-authoring.md.
```

| Previous Lesson | All Lessons | Next Lesson |
| :--- | :---: | ---: |
| [0012: Build Grafana Dashboards for Health, Traffic, Errors, and Latency](0012-grafana-dashboard-panels.md) | [Lesson index](index.md) | [0014: Transform Query Results into Grafana Summary Tables](0014-grafana-transformations-and-tables.md) |
