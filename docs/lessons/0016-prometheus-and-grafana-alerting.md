---
icon: lucide/notebook-pen
---

# 0016: Configure and Test Prometheus and Grafana Alerts

!!! note "Lesson outline"
    This page preserves the detailed session plan. Expand it into a full runnable lesson in a later session using the [authoring guide and shared lab contract](../lesson-authoring.md).

## Learning flow

``` mermaid
flowchart TD
    N0["Metric queries"]
    N1["Alert rule evaluation"]
    N2["Notification routing"]
    N3["Local firing and resolved payloads"]
    N0 --> N1
    N1 --> N2
    N2 --> N3
```

## Session plan

**Lesson file:** `0016-prometheus-and-grafana-alerting.md` · **Time:** 90–120 minutes

**Prerequisites:** 0008, 0010, and 0015.

**Outcome:** Create actionable alerts, observe state transitions, and verify local notifications end to end.

**Teach:** Prometheus `expr`/`for`/labels/annotations, inactive/pending/firing, standalone Alertmanager routing/grouping/silences; separate Grafana-managed rules, contact points, and policies. Panel thresholds alone do not notify.

**Guided lab:**

1. Write target-down and sustained-error rules with a traffic floor. Run `promtool check rules`, meaningful rule unit tests, reload, and inspect states.
2. Add standalone Alertmanager and a local webhook receiver. Inject failure; observe evaluation, pending, and routing delays plus firing/resolved payloads.
3. Create an equivalent Grafana-managed rule and local contact point; test no-data/error policies. Enable one notification path at a time.

**Verification evidence:** Rule files, passing rule tests, routing configuration, Grafana rule details, and a firing → resolved timeline with received local payloads. External email/Slack is optional later work.

**Failure / independent challenge:** Stop app, stop Prometheus, and remove a target in separate trials; explain the different results.

**Active recall:** What can `up==0` miss? Why require a traffic floor for error ratios? Who evaluates and routes each alert path?

**Official sources:** [Alerting overview](https://prometheus.io/docs/alerting/latest/overview/), [Rule tests](https://prometheus.io/docs/prometheus/latest/configuration/unit_testing_rules/), [Grafana no-data/error states](https://grafana.com/docs/grafana/latest/alerting/fundamentals/alert-rule-evaluation/nodata-and-error-states/).

``` text
$teach Expand lesson 0016 using the outline in docs/lessons/0016-prometheus-and-grafana-alerting.md and docs/lesson-authoring.md.
```

| Previous Lesson | All Lessons | Next Lesson |
| :--- | :---: | ---: |
| [0015: Build an End-to-End Golden Signals Monitoring Stack](0015-golden-signals-monitoring-project.md) | [Lesson index](index.md) | Last lesson |
