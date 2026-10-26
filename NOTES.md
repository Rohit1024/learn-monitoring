# User Learning Notes & Preferences

## Learner intent

- Collect application metrics, query them using PromQL and a UI, and set up Prometheus/Grafana dashboards.
- Wants detailed lesson plans so each lesson can be created in a separate session.
- Prior engineering, Linux, Python, and container proficiency has not been demonstrated in this workspace; check prerequisites as needed.

## Canonical curriculum

Keep `docs/lessons/index.md` as a concise index of meaningful lesson titles linking to individual pages. Each lesson page holds its own content or detailed outline; `docs/lesson-authoring.md` holds the common authoring and lab contract.

- 0001–0002: architecture, local stack, configuration, scraping.
- 0003–0005: metric types, Python/FastAPI instrumentation, app ingestion and target diagnostics.
- 0006–0010: Prometheus UI/API, PromQL selectors/time, rates/resets, aggregation/matching, latency histograms.
- 0011–0014: Grafana connections/Explore, panels, variables, transformations.
- 0015–0016: Golden Signals capstone, reproducible provisioning, local alerting.

## State as of 2026-10-26

- Lesson 0001 is authored and revised for the shared lab. This does not establish that the learner ran it.
- Lesson 0002 is authored with a shared-stack bootstrap, job timing overrides, configuration validation/reload checks, target health queries, unreachable-target and invalid-timeout exercises, evidence capture, and baseline recovery.
- Lesson 0003 is authored with pinned Python/client setup, four metric-type demonstrations, cumulative bucket interpretation, a series budget including optional creation metadata, raw-path failure/recovery, and a metric design challenge.
- Lesson 0004 is authored with complete FastAPI code, resolved dependency pins, a Dockerfile, request-accounting evidence, concurrent gauge observation, handled/unhandled error cleanup, and a deliberate gauge-leak exercise. Application files are supplied as learner-created artifacts in the lesson.
- Lesson 0005 is authored with Compose integration, application scraping, label ownership, traffic evidence, four failure/recovery exercises, and an outage challenge. Live container exercises were not run because the local engine was stopped.
- Lesson 0006 is authored with Table/Graph investigations, fixed-time instant/range API queries, controlled traffic, discovery and TSDB inspection, empty/error recovery, and a query notebook. The pinned UI/API source was checked; live container/UI exercises were not run because the engine was unavailable.
- Lessons 0007–0016 have individual detailed outline pages; full runnable lessons remain to be authored.
- Compose integration and bounded manual traffic are supplied in lesson 0005; a sustained traffic generator and provisioning files remain for later lessons.
- No learner completion or conceptual mastery has been recorded.
- Next authoring request: `$teach Expand lesson 0007 using the outline in docs/lessons/0007-promql-selectors-and-vectors.md and docs/lesson-authoring.md.`

## Teaching decisions

- Use one Python/FastAPI app with `prometheus_client`, one Uvicorn worker, and the shared metric schema in `docs/lesson-authoring.md`.
- Lesson 0003 uses Python `3.14.6`, `prometheus-client==0.26.0`, and finite latency bucket boundaries `(0.05, 0.1, 0.3, 0.5, 1.0, 2.5)` seconds, with automatic `+Inf`; carry these choices into the application lesson.
- Lesson 0004 preserves those client/bucket choices, adds FastAPI `0.162.0` and Uvicorn `0.54.0`, and wraps the complete FastAPI object to observe server-generated 500 responses. Future lessons should use its resolved requirements, one-worker command, exact `/metrics` endpoint, router-template labels, and fixed `__unmatched__` fallback.
- Keep target labels, metric names, units, and route templates consistent across PromQL and Grafana lessons.
- Explain absent series, zero traffic, scrape failure, and undefined ratios separately.
- Teach classic histogram buckets before comparing native histograms; Python Summary does not provide quantile estimates.
- Compare standalone Prometheus/Alertmanager and Grafana-managed alert paths explicitly; use local webhook notifications in the lab.
- Supply runnable prerequisites and restart notes in each authored lesson so sessions can resume without conversation history.
- Keep vertical Mermaid diagrams, expandable recall answers, top-level-only Zensical navigation, and bottom pagination.
- Record demonstrated understanding only when supported by learner responses or exercise evidence.
