---
icon: lucide/compass
---

# Mission: Application Monitoring with Prometheus & Grafana

## Why

Build an application monitoring workflow I can reproduce: expose useful metrics, collect them with Prometheus, investigate behavior with PromQL and the Prometheus UI, and communicate application health through Grafana dashboards.

## Success looks like

- Start a persistent local Prometheus/Grafana stack and explain its networking and data flow.
- Instrument a Python/FastAPI application with request counters, in-progress gauges, and latency histograms using bounded labels.
- Configure application scraping and diagnose unreachable targets, incorrect endpoints, and missing metrics.
- Write and explain PromQL selectors, counter rates, error ratios, grouped aggregations, vector matching, mean latency, and estimated percentiles.
- Investigate results in the Prometheus expression browser, reproduce queries through the HTTP API, and compare with Grafana Explore.
- Create and recreate Grafana data sources and dashboards, including appropriate units, variables, transformations, and file provisioning.
- Complete a Golden Signals lab and demonstrate local alerts firing and resolving.

## Constraints

- Five modules and sixteen sessions with detailed briefs; author future lessons in separate sessions.
- Practical sessions normally take 60–90 minutes; larger labs may take two hours.
- Use one consistent Python/FastAPI example, Docker Compose, and explicit version pins; other languages can be later adaptations.
- Every authored lesson includes a runnable starting state, vertical Mermaid diagram, guided practice, failure/recovery, active recall, verification, and restart notes.
- State prerequisites explicitly. Authored material is distinct from demonstrated learner completion.

## Out of scope

- Kubernetes operators, high-availability storage, and multi-cluster architectures in the core course.
- Full production deployment hardening; the local stack teaches the underlying monitoring workflow.
