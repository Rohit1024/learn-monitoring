---
icon: lucide/code-xml
---

# 0004: Instrument a FastAPI Application and Expose /metrics

**Time:** 90–120 minutes. **Prerequisites:** [lesson 0003](0003-metric-types-and-labels.md), a terminal, `curl`, and uv. You should recognize Python functions, `async`/`await`, and `try`/`finally`; the annotations below explain their instrumentation roles. A running container engine is required only for the container exercise.

By the end, you should be able to:

- Instrument completed requests, request duration, and requests in progress using the shared metric schema.
- Expose `/metrics` with a correct content type and bounded route labels.
- Account for successful requests, validation failures, missing routes, and application exceptions.
- Reproduce a leaked gauge and repair its cleanup path.

**Observable outcome:** A runnable four-route FastAPI application, a pinned environment and Dockerfile, and saved HTTP/metric evidence. Prometheus ingestion comes in lesson 0005; today you verify the application endpoint directly.

## 1. Follow one request through instrumentation

A **registry** collects the instruments your process exports. **Middleware** wraps application execution so you can record every request consistently. FastAPI runs on **ASGI**, an interface whose HTTP messages distinguish response start from response body. The small wrapper below reads the actual status from the response and records metrics when application execution ends.

``` mermaid
flowchart TD
    Request["HTTP request enters the metrics wrapper"]
    Exclude{"Exactly /metrics?"}
    Exposition["Export the registry without changing business request metrics"]
    Begin["Increment in-progress gauge and start monotonic timer"]
    Route["FastAPI matches a route and executes its handler"]
    Status["Capture the HTTP response status"]
    Finalize["Record count and duration using the matched route template"]
    Cleanup["Decrement the gauge in finally, including exception paths"]
    Request --> Exclude
    Exclude -->|"Yes"| Exposition
    Exclude -->|"No"| Begin
    Begin --> Route
    Route --> Status
    Status --> Finalize
    Finalize --> Cleanup
```

The three business metrics stay consistent with lesson 0003:

| Metric | Instrument labels | Meaning |
| :--- | :--- | :--- |
| `app_http_requests_total` | `method,route,status` | Requests whose application execution has ended, including errors |
| `app_http_request_duration_seconds` | `method,route` | Classic histogram of elapsed application execution time |
| `app_http_requests_in_progress` | None | Application requests currently executing |

Use `/items/{item_id}` as the label for both `/items/101` and `/items/202`. An unmatched URL uses `__unmatched__`. Query strings, raw item IDs, and exception messages do not become labels. Review [FastAPI path parameters](https://fastapi.tiangolo.com/tutorial/path-params/) and [bounded label design](https://prometheus.io/docs/practices/instrumentation/#use-labels).

## 2. Bootstrap a complete local application

From the workspace root:

``` bash
uv --version
curl --version
mkdir -p monitoring-lab/app monitoring-lab/evidence
cd monitoring-lab
uv venv --python 3.14.6 .venv-app
```

**All remaining commands run from `monitoring-lab/`.** This exercise supplies all application files, even if you have not created the lesson 0003 experiments. Keep any earlier Compose stack and volumes; the application runs separately until lesson 0005.

Save `app/requirements.txt`. These are the direct and resolved transitive dependencies used for author verification on 2026-10-24:

``` text
annotated-doc==0.0.5
annotated-types==0.8.0
anyio==4.15.1
click==8.5.0
fastapi==0.162.0
h11==0.16.0
idna==3.20
opentelemetry-api==1.45.1
prometheus-client==0.26.0
pydantic==2.14.1
pydantic-core==2.50.1
starlette==1.9.0
typing-extensions==4.16.0
typing-inspection==0.4.4
uvicorn==0.54.0
```

Install and verify:

``` bash
uv pip install --python .venv-app/bin/python --requirement app/requirements.txt
uv pip check --python .venv-app/bin/python
.venv-app/bin/python --version
.venv-app/bin/python -c 'from importlib.metadata import version; print({name: version(name) for name in ("fastapi", "uvicorn", "prometheus-client")})'
```

Expected: Python `3.14.6`, FastAPI `0.162.0`, Uvicorn `0.54.0`, client `0.26.0`, and compatible dependencies. The client and finite histogram boundaries carry forward from lesson 0003. Official release metadata: [FastAPI](https://pypi.org/project/fastapi/0.162.0/), [Uvicorn](https://pypi.org/project/uvicorn/0.54.0/), and [Python client](https://pypi.org/project/prometheus-client/0.26.0/).

## 3. Create the instruments, handlers, and middleware

Save this complete file as `app/main.py`:

``` python
import asyncio
from time import perf_counter

from fastapi import FastAPI, HTTPException, Response
from prometheus_client import (
    CONTENT_TYPE_LATEST,
    REGISTRY,
    Counter,
    Gauge,
    Histogram,
    generate_latest,
)

REQUESTS = Counter(  # (1)!
    "app_http_requests_total",
    "Completed application requests",
    ["method", "route", "status"],
)
DURATION = Histogram(
    "app_http_request_duration_seconds",
    "Application request duration in seconds",
    ["method", "route"],
    buckets=(0.05, 0.1, 0.3, 0.5, 1.0, 2.5),
)
IN_PROGRESS = Gauge(
    "app_http_requests_in_progress",
    "Application requests currently running",
)
METHODS = {"GET", "POST", "PUT", "PATCH", "DELETE", "HEAD", "OPTIONS"}
EXPECTED_GET_STATUSES = {
    "/hello": ("200", "500"),
    "/items/{item_id}": ("200", "422", "500"),
    "/slow": ("200", "500"),
    "/error": ("500",),
    "__unmatched__": ("404",),
}
for route, statuses in EXPECTED_GET_STATUSES.items():  # (2)!
    DURATION.labels(method="GET", route=route)
    for status in statuses:
        REQUESTS.labels(method="GET", route=route, status=status)

api = FastAPI(
    docs_url=None, redoc_url=None, openapi_url=None, redirect_slashes=False
)


@api.get("/hello")
async def hello():
    return {"message": "hello"}


@api.get("/items/{item_id}")
async def item(item_id: int):
    return {"item_id": item_id}


@api.get("/slow")
async def slow():
    await asyncio.sleep(2.0)  # (3)!
    return {"message": "finished slow request"}


@api.get("/error")
async def error(unhandled: bool = False):
    if unhandled:
        raise RuntimeError("Deliberate unhandled lab exception")
    raise HTTPException(status_code=500, detail="Deliberate handled lab error")


@api.get("/metrics")
async def metrics():
    return Response(
        content=generate_latest(REGISTRY),
        headers={"Content-Type": CONTENT_TYPE_LATEST},  # (4)!
    )


class MetricsMiddleware:
    def __init__(self, app):
        self.app = app

    async def __call__(self, scope, receive, send):
        if scope["type"] != "http" or scope["path"] == "/metrics":  # (5)!
            await self.app(scope, receive, send)
            return

        method = scope["method"] if scope["method"] in METHODS else "OTHER"  # (6)!
        status = "500"
        started = perf_counter()  # (7)!
        IN_PROGRESS.inc()

        async def send_with_status(message):
            nonlocal status
            if message["type"] == "http.response.start":
                status = str(message["status"])
            await send(message)

        try:
            await self.app(scope, receive, send_with_status)
        finally:
            try:
                route = getattr(scope.get("route"), "path", "__unmatched__")  # (8)!
                REQUESTS.labels(method=method, route=route, status=status).inc()
                DURATION.labels(method=method, route=route).observe(
                    perf_counter() - started
                )
            finally:
                IN_PROGRESS.dec()  # (9)!


app = MetricsMiddleware(api)  # (10)!
```

1. Create the instruments once at module import, rather than inside each request. They register with the client's default `REGISTRY`. The default registry also contains runtime collectors; business metrics are the three names beginning `app_http_`.
2. Initialize the finite GET/status combinations used by this lab, including zero error counters for the normal routes. Other bounded combinations, such as a POST returning 405, appear if observed. This is an explicit initialization policy, not every possible Cartesian combination.
3. `asyncio.sleep()` yields to the event loop, allowing a simultaneous `/metrics` request to observe the gauge. A blocking `time.sleep()` here would obstruct that observation in this single-worker example.
4. Use `generate_latest()` and its matching `CONTENT_TYPE_LATEST`; return the bytes directly rather than encoding them as JSON. This serves exactly `/metrics` with no mount redirect.
5. Pass non-HTTP ASGI scopes, such as startup/shutdown, through unchanged. Exclude `/metrics` so scrapes do not look like business traffic or increment the gauge they are inspecting.
6. Normalize unusual HTTP methods to `OTHER`, keeping this label's domain bounded. The status comes from the actual HTTP response, rather than assuming success.
7. `perf_counter()` supplies a monotonic elapsed-time measurement in seconds. Wall-clock adjustments must not create a negative request duration.
8. Read `scope["route"].path` after the router has run; use the fixed fallback if there is no route. Do not substitute `scope["path"]`, which contains raw IDs.
9. The inner `finally` releases the gauge even if execution or a metric update raises. Cleanup must not depend on whether the response was successful.
10. Wrap the whole FastAPI object. This places the metrics wrapper outside its error handling, allowing it to see the generated 500 response before an unhandled exception propagates to Uvicorn for logging.

Sources: [Starlette pure ASGI middleware and exception layers](https://starlette.dev/middleware/), [ASGI HTTP messages](https://asgi.readthedocs.io/en/latest/specs/www.html), [monotonic timing](https://docs.python.org/3.14/library/time.html#time.perf_counter), and [FastAPI exception handling](https://fastapi.tiangolo.com/tutorial/handling-errors/).

!!! note "What this duration means"
    The timer runs until the inner ASGI application returns or raises. For these ordinary JSON routes, that covers handler work, serialization, and response sending. It is server-side elapsed time, not client network latency. This lab has no streaming responses or background tasks; their completion/disconnect semantics are not part of this exercise. The same wrapper around such routes would also wait for the remainder of their ASGI execution, so do not interpret that measurement as time to response headers.

The automatic API documentation and trailing-slash redirects are disabled to keep this small route set explicit. `/error` demonstrates a handled `HTTPException`; `/error?unhandled=true` demonstrates an unexpected Python exception. Both should return HTTP 500 and release the gauge. The query parameter does not alter the route label.

!!! tip "Version-sensitive metrics integration"
    The [client's FastAPI integration guide](https://prometheus.github.io/client_python/exporting/http/fastapi-gunicorn/) also shows ASGI mounting. Its newer `add_route` example requires a client newer than our `0.26.0` pin. The explicit `Response` endpoint above uses the pinned client's exposition functions and avoids that compatibility issue.

## 4. Start one worker and capture the baseline

In terminal A, from `monitoring-lab/`:

``` bash
.venv-app/bin/python -m uvicorn main:app --app-dir app --host 127.0.0.1 --port 8000 --workers 1 --loop asyncio --http h11
```

Keep it running. Use terminal B for the remaining commands, also from `monitoring-lab/`. Do not enable `--reload` for these evidence runs; a restart resets the in-memory counters. If port 8000 is occupied by the documentation server, move that server to another port or choose a different application host port and adjust all test URLs consistently.

Before visiting business routes, inspect the metrics response:

``` bash
curl --fail --silent --show-error --dump-header evidence/0004-metrics-headers.txt --output evidence/0004-before.prom http://localhost:8000/metrics
cat evidence/0004-metrics-headers.txt
cat evidence/0004-before.prom
```

Expected: HTTP 200 and the client's text content type. For these pins, it is `text/plain; version=1.0.0; charset=utf-8`. The body includes `# HELP`, `# TYPE`, and numeric sample lines. Initialized business counters and histogram counts are zero; the in-progress gauge is zero.

You may also see `python_info`, Python GC metrics, optional `_created` samples, and platform-dependent `process_*` metrics. Those describe the runtime or creation metadata, rather than application request outcomes. Linux container process metrics can differ from a native macOS run.

## 5. Send known traffic and account for every request

Predict the count and route labels before running this batch. Use a fresh process and the baseline above if you want the exact totals; extra browser requests to business routes change the totals.

``` bash
curl --fail --silent --show-error http://localhost:8000/hello
curl --fail --silent --show-error http://localhost:8000/hello
curl --fail --silent --show-error http://localhost:8000/items/101
curl --fail --silent --show-error http://localhost:8000/items/202
curl --silent --show-error --output /dev/null --write-out '%{http_code}\n' http://localhost:8000/items/not-an-integer
curl --silent --show-error --output /dev/null --write-out '%{http_code}\n' http://localhost:8000/missing-a
curl --silent --show-error --output /dev/null --write-out '%{http_code}\n' http://localhost:8000/missing-b
curl --silent --show-error --output /dev/null --write-out '%{http_code}\n' http://localhost:8000/error
curl --fail --silent --show-error http://localhost:8000/slow
curl --fail --silent --show-error --output evidence/0004-after.prom http://localhost:8000/metrics
```

The deliberately failing requests omit `--fail` so you can inspect their HTTP status without treating the expected result as a shell failure. The last business request takes about two seconds.

??? success "Expected request accounting"

    | `route` | `status` | Completed requests |
    | :--- | :--- | ---: |
    | `/hello` | `200` | 2 |
    | `/items/{item_id}` | `200` | 2 |
    | `/items/{item_id}` | `422` | 1 |
    | `__unmatched__` | `404` | 2 |
    | `/error` | `500` | 1 |
    | `/slow` | `200` | 1 |
    | **Total** | | **9** |

    All have `method="GET"`. The item's invalid integer is a validation failure on the same matched template. Both missing URLs share the single fallback label. The gauge returns to zero. Histogram counts include failures too: `/items/{item_id}` has count three, and all route histogram counts sum to nine.

Use the client parser to summarize the two snapshots and save readable evidence:

``` bash
.venv-app/bin/python - <<'PY' > evidence/0004-summary.json
import json
from pathlib import Path
from prometheus_client.parser import text_string_to_metric_families

def samples(path):
    return [sample
            for family in text_string_to_metric_families(Path(path).read_text())
            for sample in family.samples]

before = samples("evidence/0004-before.prom")
after = samples("evidence/0004-after.prom")
name = "app_http_requests_total"
print(json.dumps({
    "completed_requests_delta": (
        sum(s.value for s in after if s.name == name)
        - sum(s.value for s in before if s.name == name)
    ),
    "nonzero_request_series": [
        {"labels": s.labels, "value": s.value}
        for s in after if s.name == name and s.value
    ],
    "in_progress": next(
        s.value for s in after if s.name == "app_http_requests_in_progress"
    ),
}, indent=2))
PY
cat evidence/0004-summary.json
```

Expected: `completed_requests_delta` is `9` and `in_progress` is `0`. The [Python parser](https://prometheus.github.io/client_python/parser/) reads exposition; this is local evidence inspection, not a Prometheus query.

Inspect the slow route's histogram. Its `le="0.3"` and `le="1.0"` buckets should remain zero for this request; `_count` and `le="+Inf"` should be one. `_sum` should be at least about two seconds. The `2.5` bucket usually includes the observation, but scheduling delays can push it higher; keep measured evidence rather than forcing a timing value. The finite boundaries remain `(0.05, 0.1, 0.3, 0.5, 1.0, 2.5)` seconds.

### Prove that scrapes do not count as business traffic

Fetch `/metrics` repeatedly:

``` bash
curl --fail --silent --show-error --output evidence/0004-scrape-one.prom http://localhost:8000/metrics
curl --fail --silent --show-error --output evidence/0004-scrape-two.prom http://localhost:8000/metrics
```

With no concurrent business requests, the `app_http_` counts, histogram observations, and gauge should be unchanged. Runtime/GC measurements may change, so do not require the entire text response to be byte-for-byte identical. These snapshots still lack the future scrape target labels `job`, `instance`, `service`, and `env`; Prometheus will attach them in lesson 0005.

## 6. Observe an in-progress request

Run one slow request in the background and inspect the gauge during its two-second pause:

``` bash
curl --fail --silent --show-error --output evidence/0004-slow-response.json http://localhost:8000/slow &
slow_request_pid=$!
sleep 0.2
curl --fail --silent --show-error --output evidence/0004-during.prom http://localhost:8000/metrics
wait "$slow_request_pid"
curl --fail --silent --show-error --output evidence/0004-finished.prom http://localhost:8000/metrics
```

Predict: the during snapshot should show `app_http_requests_in_progress 1.0`, and the finished snapshot `0.0`. If you missed the running request, repeat and capture it earlier. `/metrics` itself does not increase this gauge. After this additional request, the slow route has two completed requests, bringing the earlier total to ten.

## 7. Deliberately leak the gauge, then recover

First verify the correct application handles an unexpected exception:

``` bash
curl --silent --show-error --output evidence/0004-unhandled-body.txt --write-out '%{http_code}\n' 'http://localhost:8000/error?unhandled=true'
curl --fail --silent --show-error --output evidence/0004-unhandled.prom http://localhost:8000/metrics
```

Expected: HTTP 500; Uvicorn logs the deliberate `RuntimeError`; `/error`'s 500 counter increases once; the gauge returns to zero. Logging an exception does not mean that the worker has exited.

Now reproduce a cleanup bug:

1. Stop terminal A with ++ctrl+c++ and save the correct source:

    ``` bash
    cp app/main.py evidence/0004-main-working.py
    ```

2. In the **innermost `finally` block only**, replace its unconditional decrement with this deliberately incorrect policy:

    ``` python
    finally:
        if status != "500":
            IN_PROGRESS.dec()
    ```

3. Restart the same single-worker command from section 4. Trigger `/error?unhandled=true` once and save a new metrics response as `evidence/0004-leaked.prom`.
4. Explain why the response has ended but the gauge remains `1`. Send an ordinary `/hello` request: its increment and decrement cancel each other, so the leaked baseline remains.
5. Restore the correct source, restart the worker to clear the deliberately corrupted in-memory state, and repeat both handled and unhandled failures.

``` mermaid
flowchart TD
    Start["An error request increments the gauge to 1"]
    Error["The handler fails and FastAPI sends HTTP 500"]
    Bug["Broken cleanup skips decrement for status 500"]
    Leak["No request remains, but the gauge still reports 1"]
    Repair["Restore unconditional finally cleanup and restart the worker"]
    Repeat["Repeat the failure and verify the gauge returns to 0"]
    Start --> Error
    Error --> Bug
    Bug --> Leak
    Leak --> Repair
    Repair --> Repeat
```

??? success "Recovery commands and expected result"
    Stop the broken worker before restoring the file:

    ``` bash
    cp evidence/0004-main-working.py app/main.py
    ```

    Start terminal A again with the original command. In terminal B:

    ``` bash
    curl --silent --show-error --output /dev/null --write-out '%{http_code}\n' http://localhost:8000/error
    curl --silent --show-error --output /dev/null --write-out '%{http_code}\n' 'http://localhost:8000/error?unhandled=true'
    curl --fail --silent --show-error --output evidence/0004-recovered.prom http://localhost:8000/metrics
    ```

    Both return 500. In this fresh process, `/error`'s 500 counter and histogram count are two, and the gauge is zero. Do not reuse the earlier nine-request baseline after a restart: the counters have reset. The repair is unconditional cleanup; restarting also removes the state corrupted by the deliberate bug.

Keep the correct file as your final `app/main.py`. Record the before/failure/recovery gauge values and why the worker restart changed the counter baseline in `evidence/0004-notes.md`.

## 8. Prepare the app container

Save `app/Dockerfile`:

``` dockerfile
FROM python:3.14.6-slim
WORKDIR /app
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1
COPY requirements.txt ./
RUN python -m pip install --no-cache-dir --requirement requirements.txt
COPY main.py ./
EXPOSE 8000
CMD ["python", "-m", "uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "1", "--loop", "asyncio", "--http", "h11"]
```

This supplies a runnable container starting point for [lesson 0005](0005-scraping-application-metrics.md). See [FastAPI's container guide](https://fastapi.tiangolo.com/deployment/docker/). The base image is version-pinned; tags can still be rebuilt, so a byte-identical image also requires a digest pin.

If your container engine is running, stop the native worker to free host port 8000 and try:

``` bash
docker build --tag monitoring-demo-api:lesson0004 app
docker run --rm --name monitoring-demo-api --publish 127.0.0.1:8000:8000 monitoring-demo-api:lesson0004
```

The server binds `0.0.0.0` **inside** the container so the published host port can reach it. Repeat `/hello` and `/metrics` from another terminal. Stop the foreground container with ++ctrl+c++; `--rm` removes this temporary container. Lesson 0005 adds the service to the shared Compose network and configures scraping.

!!! note "Keep one worker"
    Each Python worker has its own in-memory registry and metric state. Increasing `--workers` without a multiprocess collection design can make scrapes see different subsets of the traffic. This lesson explicitly uses one worker; see [Uvicorn settings](https://uvicorn.dev/settings/) and [client multiprocess limitations](https://prometheus.github.io/client_python/multiprocess/).

Direct instrumentation records application events you can name in your code. An exporter instead collects and translates another system's telemetry, such as database or host metrics. Both can expose scrape endpoints; they answer different measurement questions. See [exporters and integrations](https://prometheus.io/docs/instrumenting/exporters/).

## 9. Independent challenge

Add `GET /items/{item_id}/price`, returning a fixed price for a valid integer item ID. Register it on `api`, and initialize the intended finite GET/status combinations and histogram child once at startup.

Before running it, predict what two distinct item IDs and one invalid ID will do. Verify:

- Both successful requests share `route="/items/{item_id}/price"` and increment its `200` counter twice.
- The invalid integer adds one `422` to the same template and one histogram observation.
- The gauge returns to zero, and no raw identifier appears in labels.
- Requests to two different unknown paths still use the fixed unmatched label.

Explain what the new template adds to your lesson 0003 cardinality budget. Save the final metrics snapshot and your reasoning. The four original routes remain the baseline for the next lesson; you can keep the extra bounded route if you record that addition.

## 10. Active recall

??? question "Why exclude /metrics from business request instrumentation?"
    Prometheus scrapes and manual inspections would otherwise inflate request traffic and contribute their own durations. The gauge would also include the request inspecting it.

??? question "Why read the router's template after application execution?"
    The router has matched the endpoint by then, including paths that fail parameter validation. Raw URLs contain IDs; the registered template identifies the bounded operation.

??? question "Why does a handled HTTPException differ from an unexpected RuntimeError?"
    FastAPI converts the handled exception into a normal response. The unexpected error is handled by the outer server-error layer and can then propagate for logging. The metrics wrapper must clean up in both paths and observe the actual response status.

??? question "Why use perf_counter rather than time.time for latency?"
    It measures elapsed time using a monotonic clock, avoiding jumps caused by wall-clock corrections. The observed values are seconds.

??? question "Why create instruments once rather than inside a request handler?"
    Repeated construction can register duplicate metric names or repeatedly discard state. One process should reuse the same Counter, Gauge, and Histogram objects.

??? question "Does a successful /metrics response prove the application is healthy?"
    It proves that exposition worked. The application can simultaneously be returning errors; inspect request/status metrics separately. Prometheus target scrape health is a further observation introduced in lesson 0005.

??? question "Why does a successful request not repair a previously leaked gauge?"
    Its own increment and decrement balance. They do not remove the earlier missing decrement. Fix the cleanup path and reset the deliberately corrupted lab process before validating recovery.

??? question "What belongs in labels, and what belongs in observations?"
    Bounded dimensions such as method, route template, and status belong in labels. Numeric duration belongs in histogram observations. Raw IDs and exception text do not belong in this metric schema.

## Verification and session handoff

- [ ] Install the pinned environment and start one Uvicorn worker.
- [ ] Confirm `/metrics` returns text with the client content type and initialized business metrics.
- [ ] Account for all nine guided requests using the expected route/status labels.
- [ ] Explain why repeated scrapes leave business metrics unchanged.
- [ ] Observe the gauge while `/slow` runs and after it finishes.
- [ ] Verify 422, 404, handled 500, and unhandled 500 responses are counted with bounded labels.
- [ ] Reproduce the leaked gauge, restore unconditional cleanup, restart, and verify recovery.
- [ ] Complete the new-route challenge without raw-path labels.
- [ ] Save the Dockerfile and, if an engine is available, check the foreground container.

**Author verification (2026-10-24):** Installed and checked the complete pinned dependency set with Python `3.14.6`. Live native Uvicorn checks passed for the published traffic and parser commands, scrape exclusion, concurrent gauge observations, bounded labels, handled/unhandled errors, deliberate leakage and recovery, and the extra-route challenge. The Python base-image tag was verified; the container build was not run because the local engine was stopped.

**Files created during the exercise:** `monitoring-lab/app/main.py`, `app/requirements.txt`, `app/Dockerfile`, `.venv-app/`, and `monitoring-lab/evidence/0004-*`. Record actual versions, command results, timing observations, and your failure explanation in `evidence/0004-notes.md`.

**Resume:** From `monitoring-lab/`, recreate `.venv-app` from the requirements file if needed, then run the single-worker command in section 4. **Stop:** ++ctrl+c++ in the server terminal. For earlier Compose services, `docker compose down` preserves named volumes. Application counters reset when this worker restarts; the endpoint itself stores no history.

**Next:** [Lesson 0005](0005-scraping-application-metrics.md) adds this app to Compose and scrapes `app:8000` with `job="demo-api"`, `service="demo-api"`, and `env="local"`. Keep the correct gauge cleanup, four shared routes, metric names, and bucket boundaries. There is no application scrape job to query until that integration is configured.

Authorship does not establish learner completion. Use your saved evidence and recall responses to demonstrate understanding.

``` text
$teach Guide me through lesson 0005 in docs/lessons/0005-scraping-application-metrics.md.
```

| Previous Lesson | All Lessons | Next Lesson |
| :--- | :---: | ---: |
| [0003: Choose Metric Types and Design Labels with Controlled Cardinality](0003-metric-types-and-labels.md) | [Lesson index](index.md) | [0005: Scrape Application Metrics and Troubleshoot Prometheus Targets](0005-scraping-application-metrics.md) |
