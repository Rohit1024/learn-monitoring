---
icon: lucide/notebook-pen
---

# 0015: Build an End-to-End Golden Signals Monitoring Stack

!!! note "Lesson outline"
    This page preserves the detailed session plan. Expand it into a full runnable lesson in a later session using the [authoring guide and shared lab contract](../lesson-authoring.md).

## Learning flow

``` mermaid
flowchart TD
    N0["Instrumented application and load"]
    N1["Prometheus collection and queries"]
    N2["Provisioned Grafana dashboard"]
    N3["Investigate the Golden Signals"]
    N0 --> N1
    N1 --> N2
    N2 --> N3
```

## Session plan

**Lesson file:** `0015-golden-signals-monitoring-project.md` · **Time:** 90–120 minutes

**Prerequisites:** 0001–0014, or a complete included starter snapshot.

**Outcome:** Recreate the full application → Prometheus → Grafana workflow from files and diagnose controlled incidents.

**Teach:** Traffic, errors, latency, saturation; RED metrics and capacity-based worker monitoring. Reaching capacity is different from merely receiving more requests.

**Guided lab:**

1. Assemble app, Prometheus, Grafana, and a bounded traffic generator; provision data source and dashboard files with stable UIDs.
2. Add bounded worker pool and queue metrics. Define capacity, queue, and overload behavior before choosing a saturation query.
3. Run baseline, slow, error, and overload scenarios with predictions and start/stop times; correlate graphs and raw metrics.
4. Recreate containers preserving volumes; separately document fresh-workspace bootstrap and verify dashboard provisioning.

**Verification evidence:** A runnable `monitoring-lab/`, startup README, provisioning YAML, dashboard JSON, and an incident worksheet for all four signals.

**Failure / independent challenge:** Break scraping while the API still serves traffic and distinguish this from a healthy idle API.

**Active recall:** Which metric proves queue pressure? What persists after deleting containers versus volumes? How do dashboard measurements support a diagnosis?

**Official sources:** [Golden Signals](https://sre.google/sre-book/monitoring-distributed-systems/), [Provisioning](https://grafana.com/docs/grafana/latest/administration/provisioning/).

``` text
$teach Expand lesson 0015 using the outline in docs/lessons/0015-golden-signals-monitoring-project.md and docs/lesson-authoring.md.
```

| Previous Lesson | All Lessons | Next Lesson |
| :--- | :---: | ---: |
| [0014: Transform Query Results into Grafana Summary Tables](0014-grafana-transformations-and-tables.md) | [Lesson index](index.md) | [0016: Configure and Test Prometheus and Grafana Alerts](0016-prometheus-and-grafana-alerting.md) |
