---
icon: lucide/notebook-pen
---

# 0011: Connect Grafana to Prometheus and Provision the Data Source

!!! note "Lesson outline"
    This page preserves the detailed session plan. Expand it into a full runnable lesson in a later session using the [authoring guide and shared lab contract](../lesson-authoring.md).

## Learning flow

``` mermaid
flowchart TD
    N0["Grafana Explore query"]
    N1["Provisioned Prometheus data source"]
    N2["HTTP request to Prometheus"]
    N3["Query results in Grafana"]
    N0 --> N1
    N1 --> N2
    N2 --> N3
```

## Session plan

**Lesson file:** `0011-grafana-prometheus-data-source.md` · **Time:** 60–90 minutes

**Prerequisites:** 0005 and 0008.

**Outcome:** Connect Grafana to Prometheus and reproduce a known query in Explore.

**Teach:** Server-side connection, URL, stable data source UID, default source, configured scrape interval, UI setup versus YAML provisioning.

**Guided lab:**

1. Add Prometheus through Connections/Data sources using `http://prometheus:9090`; check the connection and query `up{job="demo-api"}` in Explore.
2. Compare builder/code query modes and equivalent results in the Prometheus UI at the same evaluation time.
3. Provision `apiVersion: 1`, type `prometheus`, stable UID, URL, and 15-second interval in YAML. Avoid duplicate UI/provisioned sources; restart and verify.

**Verification evidence:** Working Explore queries and a versioned YAML file recreating the connection.

**Failure / independent challenge:** Configure `localhost:9090` inside containerized Grafana, interpret the error, and fix it.

**Active recall:** Who executes the HTTP request? Why use a stable UID in dashboard JSON? Does a Grafana interval setting change actual scraping?

**Official sources:** [Configure Prometheus](https://grafana.com/docs/grafana/latest/datasources/prometheus/configure/), [Provisioning](https://grafana.com/docs/grafana/latest/administration/provisioning/).

``` text
$teach Expand lesson 0011 using the outline in docs/lessons/0011-grafana-prometheus-data-source.md and docs/lesson-authoring.md.
```

| Previous Lesson | All Lessons | Next Lesson |
| :--- | :---: | ---: |
| [0010: Query Latency Percentiles and Thresholds from Histograms](0010-promql-histograms-and-latency.md) | [Lesson index](index.md) | [0012: Build Grafana Dashboards for Health, Traffic, Errors, and Latency](0012-grafana-dashboard-panels.md) |
