---
icon: lucide/help-circle
---

# Senior Prometheus, Grafana & Observability Interview Questions

Curated, high-signal technical and architectural interview questions for Senior Site Reliability Engineers (SRE), Platform Engineers, and Observability Architects.

---

## 🏛️ Core Architecture & TSDB Internals

- **TSDB Ingestion & Compaction Pipeline**: Walk through what happens to a sample from the moment a scrape response is received by the scrape manager until it is written to a two-hour block on disk.
- **Head Block vs Chunks**: How does Prometheus manage memory in the TSDB Head block, when does truncation happen, and what role does the Write-Ahead Log (WAL) play during unexpected restarts?
- **Pull vs Push Trade-offs**: Defend Prometheus's pull-based scraping model versus a push-based agent topology (e.g., StatsD, Datadog). When is push unavoidable and how is Pushgateway correctly applied?

---

## 🔍 PromQL & Time Series Mathematics

- **`rate()` vs `irate()` vs `increase()`**: Explain the exact mathematical difference between these functions, how counter resets are handled, and why using `irate()` in alerting rules can create flappy or dangerous conditions.
- **Vector Matching Mechanics**: Detail how one-to-one and many-to-one vector matchings work using `group_left` and `group_right`. What causes a `duplicate label set` execution error?
- **Percentile Calculation Accuracy**: How does `histogram_quantile()` approximate percentiles from bucket counters? Under what traffic distributions does it produce skewed or inaccurate results?

---

## 🚨 Alerting Engineering & SRE Best Practices

- **Multiwindow Multi-Burn-Rate Alerts**: Why do static threshold alerts fail for enterprise SLOs? Walk through the design and mathematical justification of multi-burn-rate alerts recommended by the Google SRE handbook.
- **Alertmanager De-duplication & Grouping**: How does Alertmanager prevent alert storms during widespread infrastructure outages? Explain the interaction between `group_by`, `group_wait`, `group_interval`, and `repeat_interval`.

---

## 📊 Visualization & Dashboard Engineering

- **High-Performance Grafana Design**: How do you structure Grafana dashboards to load sub-second when querying hundreds of thousands of series? Explain the role of `$__rate_interval` vs `$__interval`.
- **Dynamic Variable Chaining**: How do cascading template variables work in Grafana, and what safeguards prevent accidental quadratic query explosion against the Prometheus TSDB?

---

## 🌐 Distributed Observability & Scalability

- **Thanos vs Cortex vs Grafana Mimir**: Compare the architectural design of Thanos (sidecar-based object storage offload) with Grafana Mimir / Cortex (horizontally scalable distributor/ingestor model). What are the operational trade-offs?
- **High Cardinality Mitigation**: An engineer accidentally deploys an application that includes unique user IDs as metric labels, causing Prometheus memory to spike 5x within 30 minutes. What immediate mitigation steps do you take, and how do you prevent recurrence?
