---
icon: lucide/radio-tower
---

# 0005: Scrape Application Metrics and Troubleshoot Prometheus Targets

**Time:** 60–90 minutes. **Prerequisites:** [lesson 0002](0002-prometheus-configuration-and-scraping.md), [lesson 0004](0004-instrumenting-a-fastapi-application.md), a terminal, `curl`, and a running container engine with Docker Compose support.

By the end, you should be able to:

- Connect the instrumented application to the shared Compose network and scrape it.
- Identify which labels originate in Python and which Prometheus attaches.
- Compare endpoint output with stored samples after generating traffic.
- Distinguish transport failures, HTTP errors, slow responses, and filtered metrics.

**Observable outcome:** A healthy `demo-api` target, stored application metrics, and evidence explaining four deliberate failures and their recovery. Keep Prometheus `v3.13.5`, Grafana `13.2.4`, and the complete Python dependency pins from lesson 0004.

## 1. Trace a request to a stored sample

Recall: the application holds counters, gauges, and histograms in memory. `/metrics` serializes their current state. Prometheus periodically fetches that endpoint and stores samples; business requests do not push directly to Prometheus.

``` mermaid
flowchart TD
    Request["Client calls /items/101"]
    App["FastAPI updates instruments with route=/items/{item_id}"]
    Registry["One worker holds current metric values"]
    Endpoint["GET http://app:8000/metrics exposes those values"]
    Scrape["Prometheus scrapes every 15 seconds"]
    Labels["Attach job, instance, service, and env labels"]
    Store["Store samples with scrape timestamps"]
    Query["Query stored history through Prometheus"]
    Request --> App --> Registry --> Endpoint --> Scrape --> Labels --> Store --> Query
```

A **scrape** is one collection attempt. Its health is separate from the status of a business request: `/error` may return 500 while `/metrics` still works. Prometheus generates `up=1` for a successful scrape and `up=0` for a failed one. See [jobs, instances, and generated scrape metrics](https://prometheus.io/docs/concepts/jobs_instances/).

## 2. Restore the starting state

From the workspace root:

``` bash
mkdir -p monitoring-lab/prometheus monitoring-lab/evidence
cd monitoring-lab
docker version
docker compose version
```

**All remaining shell commands run from `monitoring-lab/`.** `docker version` must report a reachable server. If it is a Podman wrapper, use lesson 0001's engine startup instructions.

Before continuing:

1. If the shared stack files are missing, create `docker-compose.yml` and `prometheus/prometheus.yml` from the complete bootstrap in [lesson 0002, section 2](0002-prometheus-configuration-and-scraping.md#2-resume-the-shared-lab). Keep the same Compose project and named volumes.
2. If application files are missing, save the complete `app/requirements.txt`, `app/main.py`, and `app/Dockerfile` from [lesson 0004](0004-instrumenting-a-fastapi-application.md), sections 2, 3, and 8. Restore the correct unconditional gauge decrement after its failure exercise. These files supply the runnable app; no source changes are needed today.
3. Stop the foreground native Uvicorn worker or temporary lesson 0004 container with ++ctrl+c++ so host port 8000 is free.

Back up the existing files:

``` bash
cp docker-compose.yml evidence/0005-compose-before.yml
cp prometheus/prometheus.yml evidence/0005-prometheus-before.yml
```

Add this **service under the existing `services:` mapping** in `docker-compose.yml`. Keep the existing Prometheus/Grafana services, `monitoring` network, and volumes:

``` yaml
  app:
    build:
      context: ./app
    restart: unless-stopped
    ports:
      - "127.0.0.1:8000:8000"  # (1)!
    networks:
      - monitoring  # (2)!
```

1. The host uses `localhost:8000` for manual requests. The Dockerfile starts one worker listening on `0.0.0.0:8000` inside the container.
2. Prometheus and the app must share this network. Compose service discovery makes `app` resolvable there; use the service name instead of a container IP.

| Caller | Address | Purpose |
| :--- | :--- | :--- |
| Host terminal | `http://localhost:8000/metrics` | Inspect application exposition |
| Prometheus container | `http://app:8000/metrics` | Collect application metrics |
| Prometheus container | `http://localhost:9090/metrics` | Existing self-scrape |
| Host terminal | `http://localhost:9090` | Query Prometheus |

Container `localhost` refers to that container. A host port mapping is not needed for communication between services on the shared network. See [Compose networking](https://docs.docker.com/compose/how-tos/networking/).

## 3. Add the scrape job

Save this complete baseline as `prometheus/prometheus.yml`:

``` yaml
global:
  scrape_interval: 15s
  scrape_timeout: 10s
  evaluation_interval: 15s

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ["localhost:9090"]
        labels:
          service: prometheus
          env: local

  - job_name: demo-api  # (1)!
    metrics_path: /metrics
    static_configs:
      - targets: ["app:8000"]  # (2)!
        labels:
          service: demo-api  # (3)!
          env: local
```

1. Supplies `job="demo-api"`.
2. Supplies `instance="app:8000"` by default. The endpoint uses HTTP and the container port.
3. These target labels apply to scraped application series. Python still supplies only its instrument labels: `method`, `route`, `status`, and the histogram's `le` boundary.

Validate, build, and start:

``` bash
docker compose config --quiet
docker compose run --rm --no-deps --entrypoint /bin/promtool prometheus check config /etc/prometheus/prometheus.yml
docker compose up -d --build
docker compose ps
curl --fail --silent --show-error http://localhost:9090/-/ready
curl --fail --silent --show-error http://localhost:8000/hello
```

Repeat the readiness checks if startup is still in progress. If failures persist, inspect `docker compose logs --tail=60 app prometheus`. Compose configuration validation checks structure; `promtool` checks Prometheus configuration; neither proves the endpoint is reachable.

The shared stack enables `--web.enable-lifecycle`. Reload even if Prometheus was already running, then capture its active configuration:

``` bash
curl --fail --silent --show-error --request POST http://localhost:9090/-/reload
curl --fail --silent --show-error --output evidence/0005-active-config.json http://localhost:9090/api/v1/status/config
```

Confirm the active configuration contains `demo-api` and `app:8000`. A successful startup of the app alone does not reload Prometheus. See [scrape configuration](https://prometheus.io/docs/prometheus/latest/configuration/configuration/) and the [management API](https://prometheus.io/docs/prometheus/latest/management_api/).

Save this known-good configuration for every recovery below:

``` bash
cp prometheus/prometheus.yml evidence/0005-working.yml
```

## 4. Prove collection and label ownership

Allow at least one 15-second interval after readiness, then save target metadata and query scrape health:

``` bash
curl --fail --silent --show-error --output evidence/0005-targets-healthy.json 'http://localhost:9090/api/v1/targets?state=active'
curl --fail --silent --show-error --get http://localhost:9090/api/v1/query \
  --data-urlencode 'query=up{job="demo-api"}' \
  --output evidence/0005-up-healthy.json
cat evidence/0005-up-healthy.json
```

Expected: one result with value `"1"`. Inspect the `demo-api` entry in the target file: `scrapeUrl` should be `http://app:8000/metrics`, `health` should be `up`, `lastError` should be empty, and `lastScrape` should be recent. Its `labels` include the four target labels above. The [HTTP API](https://prometheus.io/docs/prometheus/latest/querying/api/) provides these observations without depending on UI menu names.

Capture exposition and stored application counters:

``` bash
curl --fail --silent --show-error --output evidence/0005-endpoint.prom http://localhost:8000/metrics
curl --fail --silent --show-error --get http://localhost:9090/api/v1/query \
  --data-urlencode 'query=app_http_requests_total{job="demo-api"}' \
  --output evidence/0005-counters-before.json
```

Find `/hello` with status `200` in both files. The raw endpoint has `method`, `route`, and `status`; the stored series additionally has `job`, `instance`, `service`, and `env`. `up` comes from Prometheus and is absent from the app's endpoint.

!!! note "A scrape samples state; it does not record every transition"
    The app's exposition has no explicit sample timestamps, so collected values receive scrape timestamps. A two-second in-progress gauge spike can occur entirely between fifteen-second scrapes. The completed-request counter still retains the increment until a process reset. Later lessons explore queries over time; a missing gauge spike does not prove no request ran.

## 5. Generate traffic and compare changes

Wait for a successful scrape before taking the preceding counter baseline. With no other traffic and no app restart, run:

``` bash
for item_id in 101 202 303; do
  curl --fail --silent --show-error "http://localhost:8000/items/$item_id"
done
curl --fail --silent --show-error http://localhost:8000/hello
curl --silent --show-error --output /dev/null --write-out '%{http_code}\n' http://localhost:8000/error
curl --fail --silent --show-error http://localhost:8000/slow
curl --fail --silent --show-error --output evidence/0005-after.prom http://localhost:8000/metrics
```

Predict the changes before querying. Wait for a successful scrape whose `lastScrape` is later than the final request, then save:

``` bash
curl --fail --silent --show-error --get http://localhost:9090/api/v1/query \
  --data-urlencode 'query=app_http_requests_total{job="demo-api"}' \
  --output evidence/0005-counters-after.json
```

??? success "Expected changes from the captured baseline"
    `/items/{item_id}` status `200` increases by three, `/hello` status `200` by one, `/error` status `500` by one, and `/slow` status `200` by one. Total increase: six. Compare deltas, not absolute values: section 3 already called `/hello`. No raw item ID should appear as a route label.

    The histogram counts also increase by six in total. The gauge returns to zero. The `/error` response is 500, but the next successful metrics scrape still has `up=1`.

Use the same API command with each of these expressions, or enter them in Prometheus's query page at `http://localhost:9090`:

``` promql
app_http_request_duration_seconds_count{job="demo-api"}
```

``` promql
app_http_requests_in_progress{job="demo-api"}
```

Repeated `/metrics` requests should not change the business counts. If raw and stored values differ, check scrape timing, active configuration, concurrent traffic, and process restarts before changing instrumentation.

## 6. Break collection, identify the layer, and recover

Change only one condition at a time. For each experiment, save target metadata using section 4's command with a distinct filename such as `evidence/0005-targets-port.json`. Record the endpoint, `lastError`, and `up` value in `evidence/0005-notes.md`.

After **every configuration edit**, run:

``` bash
docker compose exec prometheus promtool check config /etc/prometheus/prometheus.yml
curl --fail --silent --show-error --request POST http://localhost:9090/-/reload
```

Only reload after validation succeeds. Wait for a new scrape attempt before interpreting its result. To restore the known-good baseline after each experiment:

``` bash
cp evidence/0005-working.yml prometheus/prometheus.yml
docker compose exec prometheus promtool check config /etc/prometheus/prometheus.yml
curl --fail --silent --show-error --request POST http://localhost:9090/-/reload
```

Verify `up{job="demo-api"}` returns one before moving on.

### A. Wrong port

Change only `app:8000` to `app:8001` in the `demo-api` job. Validate and reload. Expect a failed scrape, normally a connection-refused error because nothing listens there. The current target's `instance` changes to `app:8001`; inspect that exact target rather than an old series. Restore the baseline.

### B. Wrong path

Change only the app job's `metrics_path` to `/missing-metrics`. Validate and reload. Expect HTTP 404 and `up=0`. The network connection worked; the requested resource did not. Restore the baseline. This invalid scrape path is a normal unmatched request to the app, so it can increment the `__unmatched__` business counter.

### C. Timeout

Under `job_name: demo-api`, add `scrape_timeout: 1s` and change `metrics_path` to `/slow`. Validate and reload. This route waits about two seconds before responding, so expect a timeout such as `context deadline exceeded`. The target is reachable but does not respond within the deadline. Restore the baseline.

!!! tip "Increasing the timeout is not the repair for this experiment"
    `/slow` returns JSON, not metrics exposition. A longer timeout would reveal another problem rather than make it a valid metrics endpoint. Restore `/metrics` and the ten-second timeout. Allow outstanding slow requests to finish before checking the gauge.

``` mermaid
flowchart TD
    Failure["Scrape failed: read the current target's lastError"]
    Layer{"Which failure was observed?"}
    Port["Connection refused: check address, listener, and network"]
    Path["HTTP 404: check metrics_path"]
    Deadline["Timeout: check endpoint work and scrape deadline"]
    Repair["Restore known-good configuration and reload"]
    Verify["Wait for a fresh scrape and verify up=1"]
    Failure --> Layer
    Layer -->|"Transport"| Port
    Layer -->|"HTTP"| Path
    Layer -->|"Deadline"| Deadline
    Port --> Repair
    Path --> Repair
    Deadline --> Repair
    Repair --> Verify
```

### D. Valid scrape, missing custom metric

Starting from the working configuration, add this block under `job_name: demo-api`, aligned with `metrics_path` and `static_configs`:

``` yaml
    metric_relabel_configs:
      - source_labels: [__name__]
        regex: app_http_requests_total
        action: drop
```

Validate and reload. After a fresh successful scrape, compare:

``` promql
up{job="demo-api"}
```

``` promql
app_http_requests_total{job="demo-api"}
```

``` promql
scrape_samples_scraped{job="demo-api"}
```

``` promql
scrape_samples_post_metric_relabeling{job="demo-api"}
```

Expected: `up` stays one, the current counter selector becomes empty once the dropped series are marked stale, and the post-relabel sample count is smaller. Fetch `/metrics` directly: the counter still exists there. This filter drops the exact `_total` sample name, not its optional `_created` companion. Historical samples remain stored; an empty current selector does not erase the past.

**Target relabeling** (`relabel_configs`) acts before collection and can change or discard targets. **Metric relabeling** (`metric_relabel_configs`) acts on collected samples before ingestion. It does not filter generated `up`. See [relabeling configuration](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#metric_relabel_configs).

Restore the baseline and verify the counter returns after a fresh scrape. If a metric is missing without a filter, inspect the endpoint, label initialization, spelling, and query selectors too.

## 7. Independent challenge: app outage and recovery

With the correct configuration active:

``` bash
docker compose stop app
```

Keep Prometheus running. Wait for the next scrape attempt and save target metadata and the `up` query. Depending on the container engine and name resolution, the error may mention connection refusal or DNS failure; classify the actual evidence rather than assuming exact wording.

Recover:

``` bash
docker compose up -d app
curl --fail --silent --show-error http://localhost:8000/hello
```

Repeat the request once the app is ready if startup was still in progress. After a successful scrape, save recovery evidence. Explain:

1. Why did `up` change even though you never defined an `up` instrument in Python?
2. Why can the app's counter now be lower while Prometheus still retains earlier samples?
3. Why is stopping the app different from removing its target from configuration?
4. Which checks would you perform if the host endpoint works but the scrape still fails?

??? success "Reasoning checkpoint"
    Prometheus records scrape success independently of the app. Restarting the single worker resets its in-memory instruments; the Prometheus volume retains history. Keeping a configured but failing target produces failed-scrape observations. Removing a target ends collection for it rather than continually recording failure. A working host URL does not prove the service address, container listener, shared network, path, or active scrape configuration is correct.

Optional extension: read [file-based service discovery](https://prometheus.io/docs/guides/file-sd/) and describe how moving the target list to a file differs from changing metric labels. Keep static discovery as the final state for this course.

## 8. Active recall

??? question "Where do service and env come from?"
    The target's static labels in Prometheus configuration. Python supplies the bounded request dimensions; it does not repeat deployment labels in every instrument.

??? question "Why is localhost:8000 wrong inside the Prometheus container?"
    It points at Prometheus's own container. The app is a different service, reached as app:8000 on the shared network.

??? question "Can up=1 coexist with HTTP 500 business responses or missing counters?"
    Yes. It reports scrape success. The business route may fail independently, and a metric may be absent, uninitialized, filtered, or excluded by the query.

??? question "Does a successful reload establish reachability?"
    No. It establishes that configuration was accepted. A subsequent scrape supplies reachability and exposition evidence.

??? question "Why might a stored counter lag the endpoint?"
    A request can finish after the most recent scrape. Compare the endpoint with a fresh successful scrape and account for concurrent requests and resets.

??? question "Will fifteen-second scraping capture every two-second gauge spike?"
    No. Gauges represent sampled state. Completed-request counters retain increments between scrapes unless the process resets.

## Verification and session handoff

- [ ] Build the lesson 0004 app and join it to the shared Compose network.
- [ ] Validate configuration, reload, and inspect the active job and target.
- [ ] Capture `up=1` and all four target labels on application series.
- [ ] Demonstrate the six-request counter delta with bounded route labels.
- [ ] Explain why business errors do not imply scrape failure.
- [ ] Capture and repair the wrong-port, wrong-path, timeout, and metric-filter experiments.
- [ ] Stop/restart the app and explain scrape recovery and counter resets.
- [ ] Restore `evidence/0005-working.yml`, verify collection, and write a request-to-storage trace in `evidence/0005-notes.md`.

**Files changed during the exercise:** `monitoring-lab/docker-compose.yml`, `monitoring-lab/prometheus/prometheus.yml`, and evidence under `monitoring-lab/evidence/0005-*`. Application files and dependency pins come from lesson 0004. Final state: self-scrape plus `demo-api` at `app:8000/metrics`, fifteen-second interval, ten-second timeout, and no metric filter.

**Resume:** From `monitoring-lab/`, run `docker compose up -d --build`, check readiness and target metadata, then query `up{job="demo-api"}`. **Stop:** `docker compose down` preserves named volumes. App metric state still resets when its process restarts.

**Author verification (2026-10-26):** Official configuration, networking, target-health, and API documentation reviewed. Documentation and diagram checks are reported with the authoring result. Live container exercises were not run because the local container engine was stopped; their observations above are expected results for the learner to verify.

**Next:** [Lesson 0006](0006-prometheus-ui-and-query-api.md) develops the expression-browser and HTTP API workflow using this healthy application target. Authored material does not establish learner completion.

``` text
$teach Guide me through lesson 0006 in docs/lessons/0006-prometheus-ui-and-query-api.md.
```

| Previous Lesson | All Lessons | Next Lesson |
| :--- | :---: | ---: |
| [0004: Instrument a FastAPI Application and Expose /metrics](0004-instrumenting-a-fastapi-application.md) | [Lesson index](index.md) | [0006: Explore Metrics in the Prometheus UI and HTTP Query API](0006-prometheus-ui-and-query-api.md) |
