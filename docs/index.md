---
icon: lucide/activity
---

# Application Monitoring with Prometheus and Grafana

Welcome to your hands-on learning workspace for **Monitoring with Prometheus and Grafana**.

This curriculum guides you from a local Prometheus/Grafana stack to application instrumentation, PromQL investigations in the Prometheus UI, and reusable Grafana dashboards. One Python/FastAPI application connects the exercises across the course.

The [lesson index](lessons/index.md) links to **16 lessons in 5 modules**. Lessons 0001–0006 are authored; each remaining page contains its own detailed outline with prerequisites, exercises, verification evidence, and a request for expanding it in a separate session.

---

## 🎯 Quick Navigation

- 🧭 [**The Mission**](mission.md) — Practical goals, learning outcomes, and scope.
- 🎓 [**Curriculum & Lessons**](lessons/index.md) — 5 focused modules structured for separate, hands-on learning sessions.
- ⚡ [**Cheatsheets**](cheatsheet/index.md) — Fast PromQL functions, scrape configs, metric types, and query syntax.
- 🔍 [**Debugging Guides**](debugging/index.md) — Step-by-step diagnostic workflows for scrape failures, missing metrics, and dashboard errors.
- 💼 [**Interview Questions**](interview/index.md) — High-signal technical and system design questions for Prometheus and Grafana.
- 📚 [**References & Glossary**](references/index.md) — Authoritative CNCF specifications, Prometheus/Grafana documentation, and canonical terms.

---

## 🏛️ Course Roadmap

``` mermaid
flowchart TD
    M1["1. Prometheus & Grafana Setup & Architecture"] --> M2["2. Application Instrumentation & Metric Collection"]
    M2 --> M3["3. PromQL Query Language & Web UI Exploration"]
    M3 --> M4["4. Grafana Dashboards & Visualizations"]
    M4 --> M5["5. End-to-End Golden Signals & Alerting"]
```

---

## 🚀 Get Started Immediately

Begin with [lesson 0001](lessons/0001-setting-up-prometheus-and-grafana.md), continue through the authored lessons up to [lesson 0006](lessons/0006-prometheus-ui-and-query-api.md), or expand the next outline in a new session:

``` text
$teach Expand lesson 0007 using the outline in docs/lessons/0007-promql-selectors-and-vectors.md and docs/lesson-authoring.md.
```
