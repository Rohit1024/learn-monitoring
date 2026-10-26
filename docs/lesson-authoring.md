# Lesson Authoring and Shared Monitoring Lab

Use this guide when expanding a lesson outline into a full lesson in a separate session. Read [the mission](mission.md), the selected [lesson page](lessons/index.md), `NOTES.md`, and [official resources](references/resources.md) before authoring.

Lessons 0001–0006 are authored. Lessons 0007–0016 have individual outline pages ready to expand; page existence does not establish learner completion.

## Common authoring contract


1. Begin with objectives, prerequisites, estimated time, and an observable outcome. Supply a short recap and a runnable starting state: complete files or an explicit bootstrap procedure for prerequisite artifacts. Self-contained lessons still have conceptual prerequisites.
2. Explain the mental model using a vertical Mermaid diagram. Introduce vocabulary before using it; annotate important code/configuration lines.
3. Give exact filenames, working directories, pinned dependencies, commands, and expected observations. Distinguish expected output from results actually verified by the author.
4. Include a guided lab, one deliberate failure and recovery, an independent challenge, and at least three active-recall questions with expandable answers. Ask for predictions before revealing results.
5. Finish with a verification checklist and restart/handoff block: files changed, versions, startup/stop commands, useful queries, and next prerequisites. Preserve named volumes when stopping the lab.
6. Link official sources and recheck version-sensitive UI steps. Keep `docs/lessons/index.md` limited to meaningful lesson titles linked to their individual pages. Put outlines and lesson content on those pages, update adjacent pagination, and keep sidebar navigation at section roots.
7. Run Mermaid validation and the Zensical build as required by the teach skill. Record demonstrated understanding only after the learner supplies evidence.

## Shared lab and metric contract

Use **Python/FastAPI with `prometheus_client`** and one Uvicorn worker for the main application. Other languages can be later adaptations. Increasing worker count requires an explicit multi-process metric collection strategy.

These are target artifact paths; future sessions create application and provisioning files as introduced:

``` text
monitoring-lab/
├── docker-compose.yml
├── app/                         # API code, pinned dependencies, Dockerfile
├── prometheus/
│   ├── prometheus.yml
│   └── rules/                   # Lesson 0016
├── grafana/
│   ├── provisioning/datasources/
│   ├── provisioning/dashboards/
│   └── dashboards/
├── load/                        # Bounded traffic generator
└── evidence/                    # Queries, observations, restart notes
```

- Host URLs: Prometheus `http://localhost:9090`, Grafana `http://localhost:3000`, application `http://localhost:8000`.
- Inside Compose, scrape `app:8000` and configure Grafana with `http://prometheus:9090`. Container `localhost` refers to that container.
- Begin with 15-second scraping. Generate several minutes of traffic for five-minute query windows; counter rates require at least two samples per series.
- Add target labels `job="demo-api"`, `service="demo-api"`, and `env="local"`. Prometheus supplies `instance`. Add `service` and `env` in scrape configuration rather than repeating them in every client metric definition.
- Provide normal `/hello`, parameterized `/items/{item_id}`, slow `/slow`, and failing `/error` routes. Label by route template, excluding `/metrics` from business request metrics.
- Use classic histograms initially so bucket series are visible; compare native histograms after the classic model is understood.
- Lesson 0003 pins Python `3.14.6` and `prometheus-client==0.26.0` for isolated experiments. Preserve the client pin when introducing the application unless a later lesson explicitly explains an update.
- Lesson 0004 adds FastAPI `0.162.0`, Uvicorn `0.54.0`, Starlette `1.9.0`, and Pydantic `2.14.1`, with the complete resolved dependency list in its `app/requirements.txt` example. Its ASGI wrapper records application execution through return/exception, uses the matched route template and `__unmatched__` fallback, and excludes exactly `/metrics`.

Keep these application metrics consistent across lessons:

- `app_http_requests_total{method,route,status}`: completed requests, including handled errors. Preinitialize the finite combinations used in the lab to distinguish zero errors from absent series.
- `app_http_request_duration_seconds{method,route}`: classic histogram, exposing `_bucket{le}`, `_sum`, and `_count`. Lesson 0003 chooses finite boundaries `(0.05, 0.1, 0.3, 0.5, 1.0, 2.5)` seconds; the client adds `+Inf`. Preserve the 0.3-second boundary for the threshold exercise.
- `app_http_requests_in_progress`: gauge returning to zero after traffic stops, including error paths.
- Capstone additions: `app_work_queue_depth`, `app_workers_busy`, and `app_workers_capacity` for a bounded worker pool. Measure saturation against defined capacity; request volume alone is insufficient.

Use finite labels and base units; avoid user IDs, request IDs, and raw URLs. References: [metric naming](https://prometheus.io/docs/practices/naming/), [Python instrumentation](https://prometheus.github.io/client_python/instrumenting/), and [Compose networking](https://docs.docker.com/compose/how-tos/networking/).

## Completion evidence

The course is complete when you can recreate the stack, instrument a route without unbounded labels, diagnose a failed scrape, explain rate/error/latency queries, recreate a filtered dashboard from files, and demonstrate a local alert firing and resolving. Keep your query notebook and incident worksheet as evidence; authorship alone does not establish mastery.
