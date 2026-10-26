# Prometheus & Grafana Resources

Official sources for the [16-session curriculum](../lessons/index.md). Technical guidance was checked against these documentation families on **2026-10-23**. Links containing `latest` can change; lesson authors should match UI instructions and available features to their pinned versions.

## Setup and scraping — lessons 0001–0002

- [Prometheus overview](https://prometheus.io/docs/introduction/overview/): component responsibilities and the pull model.
- [Prometheus installation](https://prometheus.io/docs/prometheus/latest/installation/): official images, startup flags, storage, and configuration mounts.
- [Prometheus configuration](https://prometheus.io/docs/prometheus/latest/configuration/configuration/): scrape jobs, intervals, timeouts, labels, relabeling, and reload behavior.
- [Jobs and instances](https://prometheus.io/docs/concepts/jobs_instances/): target identity and scrape health metrics.
- [Management API](https://prometheus.io/docs/prometheus/latest/management_api/): readiness, health, and lifecycle endpoints.
- [Prometheus v3.13.5 configuration implementation](https://github.com/prometheus/prometheus/blob/v3.13.5/config/config.go): version-specific validation of explicit scrape timeouts in lesson 0002.
- [Prometheus v3.13.5 scrape lifecycle](https://github.com/prometheus/prometheus/blob/v3.13.5/scrape/scrape.go): delayed staleness markers and scrape-pool shutdown behavior in lesson 0002.
- [Docker Compose networking](https://docs.docker.com/compose/how-tos/networking/): service discovery and container/host addresses.
- [Prometheus releases](https://prometheus.io/download/) and [Grafana releases](https://grafana.com/grafana/download): version pin selection. Lesson 0001 uses Prometheus `v3.13.5` and Grafana `13.2.4`.

## Instrumentation — lessons 0003–0005

- Lesson 0005 rechecked the official scrape configuration, Compose networking, jobs/instances, and HTTP API references on **2026-10-26**.
- [File-based service discovery](https://prometheus.io/docs/guides/file-sd/): optional extension after static application scraping.

- Lesson 0003 sources below were rechecked on **2026-10-24**. Its isolated lab uses Python `3.14.6` and `prometheus-client==0.26.0`.
- [Prometheus data model](https://prometheus.io/docs/concepts/data_model/): series identity and the effect of changing labels.
- [Metric types](https://prometheus.io/docs/concepts/metric_types/): Counter, Gauge, Histogram, and Summary.
- [Metric and label naming](https://prometheus.io/docs/practices/naming/): units, suffixes, and dimensional design.
- [Python client instrumentation](https://prometheus.github.io/client_python/instrumenting/): the course's client library.
- [Python Counter](https://prometheus.github.io/client_python/instrumenting/counter/), [Gauge](https://prometheus.github.io/client_python/instrumenting/gauge/), and [labels](https://prometheus.github.io/client_python/instrumenting/labels/): update methods, reset behavior, and child initialization.
- [Python Histogram](https://prometheus.github.io/client_python/instrumenting/histogram/) and [Python Summary](https://prometheus.github.io/client_python/instrumenting/summary/): client-specific behavior; do not assume Summary provides quantile estimates.
- [Histograms and summaries](https://prometheus.io/docs/practices/histograms/): distribution aggregation and quantile tradeoffs.
- [Instrumentation label guidance](https://prometheus.io/docs/practices/instrumentation/#use-labels): bounded dimensions and cardinality costs.
- [Python client v0.26.0 release](https://pypi.org/project/prometheus-client/0.26.0/) and [pinned metric implementation](https://github.com/prometheus/client_python/blob/v0.26.0/prometheus_client/metrics.py): the experiment pin and optional `_created` series.
- [uv environments](https://docs.astral.sh/uv/pip/environments/): isolated Python environments for the lesson 0003 experiments.
- [FastAPI integration](https://prometheus.github.io/client_python/exporting/http/fastapi-gunicorn/): metrics exposition integration; keep the initial lab to one worker.
- Lesson 0004 sources were checked on **2026-10-24** against its resolved Python `3.14.6`, FastAPI `0.162.0`, Uvicorn `0.54.0`, and client `0.26.0` environment. The integration guide's newer `add_route` example requires a later client; lesson 0004 instead serves `generate_latest()` bytes in an explicit response.
- [FastAPI path parameters](https://fastapi.tiangolo.com/tutorial/path-params/) and [error handling](https://fastapi.tiangolo.com/tutorial/handling-errors/): route templates, validation failures, and handled exceptions.
- [Starlette middleware](https://starlette.dev/middleware/) and the [ASGI HTTP specification](https://asgi.readthedocs.io/en/latest/specs/www.html): middleware placement, response status messages, and exception cleanup.
- [Python monotonic performance counter](https://docs.python.org/3.14/library/time.html#time.perf_counter): elapsed-time measurement in seconds.
- [Python exposition](https://prometheus.github.io/client_python/exporting/http/), [parser](https://prometheus.github.io/client_python/parser/), and [multiprocess mode](https://prometheus.github.io/client_python/multiprocess/): response formats, evidence inspection, and worker limitations.
- [Uvicorn settings](https://uvicorn.dev/settings/) and [FastAPI containers](https://fastapi.tiangolo.com/deployment/docker/): the single-worker launch command and Dockerfile.
- [FastAPI `0.162.0`](https://pypi.org/project/fastapi/0.162.0/) and [Uvicorn `0.54.0`](https://pypi.org/project/uvicorn/0.54.0/): application dependency pins.
- [Exporters and integrations](https://prometheus.io/docs/instrumenting/exporters/): collecting metrics from systems without modifying their application code.

## PromQL and investigation — lessons 0006–0010

- Lesson 0006 checks the [v3.13.5 UI routes](https://github.com/prometheus/prometheus/blob/v3.13.5/web/ui/mantine-ui/src/App.tsx), [query controls](https://github.com/prometheus/prometheus/blob/v3.13.5/web/ui/mantine-ui/src/pages/query/QueryPanel.tsx), and [pinned API specification](https://github.com/prometheus/prometheus/blob/v3.13.5/docs/querying/api.md). The pinned query page uses `/query`; the general expression-browser guide still mentions `/graph`.

- [Expression browser](https://prometheus.io/docs/visualization/browser/): ad-hoc querying in the built-in UI.
- [Querying basics](https://prometheus.io/docs/prometheus/latest/querying/basics/): types, selectors, matching, evaluation time, and range queries.
- [Query functions](https://prometheus.io/docs/prometheus/latest/querying/functions/): counter functions, reset handling, estimated increases, and histogram quantiles.
- [Operators](https://prometheus.io/docs/prometheus/latest/querying/operators/): aggregation, arithmetic, and vector matching.
- [HTTP API](https://prometheus.io/docs/prometheus/latest/querying/api/): instant/range query parameters and JSON result types.
- [Storage](https://prometheus.io/docs/prometheus/latest/storage/): local storage, WAL, retention, and persistence limits.

## Grafana — lessons 0011–0014

- [Configure the Prometheus data source](https://grafana.com/docs/grafana/latest/datasources/prometheus/configure/): connection and interval settings.
- [Prometheus query editor](https://grafana.com/docs/grafana/latest/datasources/prometheus/query-editor/): builder/code modes, query types, and inspection.
- [Visualizations](https://grafana.com/docs/grafana/latest/visualizations/panels-visualizations/visualizations/): panel types and their intended use.
- [Prometheus template variables](https://grafana.com/docs/grafana/latest/datasources/prometheus/template-variables/): Label values editor, multi-select matching, and interval variables. Legacy `label_values(...)` is Grafana syntax, not PromQL.
- [Transform data](https://grafana.com/docs/grafana/latest/visualizations/panels-visualizations/query-transform-data/transform-data/): joins, calculations, frame organization, and transformation order.
- [Provisioning](https://grafana.com/docs/grafana/latest/administration/provisioning/): data sources and dashboard files.

## Integration and alerts — lessons 0015–0016

- [Google SRE: Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/): Golden Signals and actionable monitoring.
- [Prometheus alerting overview](https://prometheus.io/docs/alerting/latest/overview/): rule evaluation and standalone Alertmanager responsibilities.
- [Alerting rules](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/) and [rule unit tests](https://prometheus.io/docs/prometheus/latest/configuration/unit_testing_rules/): rule syntax and time-based verification.
- [Grafana Alerting fundamentals](https://grafana.com/docs/grafana/latest/alerting/fundamentals/): Grafana-managed alerting path.
- [Grafana no-data and error states](https://grafana.com/docs/grafana/latest/alerting/fundamentals/alert-rule-evaluation/nodata-and-error-states/): intentional handling of missing data and evaluation failures.

## Optional deeper reading and communities

- [Prometheus: Up & Running](https://www.oreilly.com/library/view/prometheus-up/9781492034131/): supplemental book; check version-sensitive details against official documentation.
- [Robust Perception](https://www.robustperception.io/blog/): practitioner explanations.
- [Grafana Community Forums](https://community.grafana.com/) and [Prometheus Users](https://groups.google.com/g/prometheus-users): discussion and troubleshooting; verify technical conclusions against primary sources.
- [Thanos documentation](https://thanos.io/tip/thanos/quick-tutorial.md/): optional material beyond the core local monitoring course.

| Previous | References | Next |
| :--- | :---: | ---: |
| [Reference index](index.md) | [All references](index.md) | [Glossary](glossary.md) |
