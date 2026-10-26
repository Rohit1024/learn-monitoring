---
icon: lucide/shapes
---

# 0003: Choose Metric Types and Design Labels with Controlled Cardinality

**Time:** 60–90 minutes. **Prerequisites:** the metrics pipeline from [lesson 0001](0001-setting-up-prometheus-and-grafana.md), a terminal, [uv](https://docs.astral.sh/uv/), and enough Python to run a script and follow a loop. Lesson 0002 is useful for target labels. No instrumented application or running container engine is required for these file-based experiments.

By the end, you should be able to:

- Choose a Counter, Gauge, Histogram, or Summary from the question you want to answer.
- Read the sample families emitted by the Python client, including cumulative histogram buckets.
- Use bounded label domains and estimate series counts before adding a metric.
- Reproduce a raw-path cardinality problem and repair it with a route template.

**Observable outcome:** Produce metric snapshots and a design sheet for the shared application's three metric families, including units, label domains, bucket boundaries, and a series budget. Use **Python `3.14.6`** and **`prometheus-client==0.26.0`**, checked against the [official package release](https://pypi.org/project/prometheus-client/0.26.0/) on 2026-10-24.

## 1. Choose the measurement before the instrument

Recall that Prometheus stores samples over time. A series is identified by its metric name and labels; changing a label value creates another series. Repeated observations with the same name and labels update the existing series. See the [data model](https://prometheus.io/docs/concepts/data_model/).

``` mermaid
flowchart TD
    Question["State the operational question"]
    Behavior["Decide whether the value accumulates, varies, or describes a distribution"]
    Instrument["Choose Counter, Gauge, Histogram, or Summary"]
    Schema["Choose a metric name, base unit, and bounded labels"]
    Budget["Count label combinations and emitted sample families"]
    Verify["Run the experiment and compare exposition with the budget"]
    Question --> Behavior
    Behavior --> Instrument
    Instrument --> Schema
    Schema --> Budget
    Budget --> Verify
```

Before reading the choices, decide how you would measure completed requests, requests currently running, and request latency.

| Question | Instrument | How the value changes | Course example |
| :--- | :--- | :--- | :--- |
| How many requests have completed? | Counter | Accumulates; may reset when its process restarts | `app_http_requests_total` |
| How many requests are in progress now? | Gauge | Increases and decreases | `app_http_requests_in_progress` |
| How are request durations distributed? | Classic Histogram | Observations update cumulative buckets, sum, and count | `app_http_request_duration_seconds` |
| What are the observation count and sum? | Python Summary | Observations update count and sum; no quantiles | `lab_request_duration_seconds` |

Sources: [core metric types](https://prometheus.io/docs/concepts/metric_types/), [Python Counter](https://prometheus.github.io/client_python/instrumenting/counter/), [Gauge](https://prometheus.github.io/client_python/instrumenting/gauge/), [Histogram](https://prometheus.github.io/client_python/instrumenting/histogram/), and [Summary](https://prometheus.github.io/client_python/instrumenting/summary/).

!!! note "An instrument can emit several series"
    A classic histogram is one instrument in your code, but its buckets, sum, and count are separate float-valued series in Prometheus. Native histogram samples have a different representation. This lesson uses classic histograms; count the emitted series rather than just the number of constructors.

Use names with one clear quantity and base unit: seconds for duration, bytes for size, and `_total` for an accumulating count. Keep durations as numbers passed to `.observe()`, rather than label values. See [metric and label naming](https://prometheus.io/docs/practices/naming/).

## 2. Create an isolated experiment directory

From the workspace root:

``` bash
uv --version
mkdir -p monitoring-lab/experiments monitoring-lab/evidence
cd monitoring-lab
uv venv --python 3.14.6 .venv-metrics
```

**All remaining shell commands run from `monitoring-lab/`.** uv can obtain the pinned Python version if necessary. Use a new environment named `.venv-metrics` so these exercises have their own dependencies.

Save `experiments/requirements-metrics.txt`:

``` text
prometheus-client==0.26.0
```

Install and record the versions:

``` bash
uv pip install --python .venv-metrics/bin/python --requirement experiments/requirements-metrics.txt
.venv-metrics/bin/python --version
.venv-metrics/bin/python -c 'from importlib.metadata import version; print(version("prometheus-client"))'
```

Expected: Python `3.14.6` and client `0.26.0`. If installation fails, resolve the package-index or certificate error before continuing; an import error is an environment problem, not a metric-type result. uv's [environment guide](https://docs.astral.sh/uv/pip/environments/) explains this setup.

The shared Compose files can stay as they are. Today's scripts write exposition files and exit. They do not expose an HTTP endpoint or send samples to Prometheus; ingestion comes in later lessons.

## 3. Exercise all four metric types

Save the complete script as `experiments/metric_types.py`:

``` python
import argparse
from pathlib import Path

from prometheus_client import (
    CollectorRegistry,
    Counter,
    Gauge,
    Histogram,
    Summary,
    disable_created_metrics,
    enable_created_metrics,
    generate_latest,
)

parser = argparse.ArgumentParser()
parser.add_argument("--without-created", action="store_true")
args = parser.parse_args()
if args.without_created:
    disable_created_metrics()
else:
    enable_created_metrics()  # (1)!

registry = CollectorRegistry()  # (2)!
requests = Counter(
    "app_http_requests_total",
    "Completed application requests",
    ["method", "route", "status"],
    registry=registry,
)
in_progress = Gauge(
    "app_http_requests_in_progress",
    "Application requests currently running",
    registry=registry,
)
duration = Histogram(
    "app_http_request_duration_seconds",
    "Application request duration in seconds",
    ["method", "route"],
    buckets=(0.05, 0.1, 0.3, 0.5, 1.0, 2.5),  # (3)!
    registry=registry,
)
summary = Summary(
    "lab_request_duration_seconds",  # (4)!
    "Comparison observations for this experiment",
    registry=registry,
)

for status in ("200", "500"):
    requests.labels(method="GET", route="/hello", status=status)  # (5)!
latency = duration.labels(method="GET", route="/hello")
evidence = Path("evidence")
evidence.mkdir(exist_ok=True)

for index, seconds in enumerate((0.04, 0.12, 0.5, 1.2)):
    in_progress.inc()
    try:
        if index == 0:
            (evidence / "0003-in-progress.prom").write_bytes(
                generate_latest(registry)
            )
        requests.labels(method="GET", route="/hello", status="200").inc()
        latency.observe(seconds)  # (6)!
        summary.observe(seconds)
    finally:
        in_progress.dec()  # (7)!

print(generate_latest(registry).decode("utf-8"), end="")
```

1. Enable the client's usual `_created` output explicitly, making the default experiment independent of any inherited disabling environment variable. The flag lets you compare both modes.
2. A registry collects metric objects. A private registry keeps this experiment's output free of the default process/Python collectors. Create each instrument once, before recording observations.
3. These are six finite boundaries in **seconds**. The client also supplies `+Inf`. Keep the `0.3` boundary for later threshold exercises.
4. This comparison Summary belongs only to the experiment. The shared application will use the Histogram for latency.
5. Calling `.labels()` creates a zero-valued child before traffic. This example initializes `200` and `500`; it does not initialize `404`.
6. The observations are fixed synthetic durations. The loop does not sleep or measure actual elapsed request time.
7. Cleanup returns the gauge to zero even if the block raises an exception. The first snapshot captures one request in progress.

Before running it, predict the successful request count, error count, final gauge, and histogram count:

``` bash
.venv-metrics/bin/python experiments/metric_types.py > evidence/0003-types.prom
cat evidence/0003-types.prom
cat evidence/0003-in-progress.prom
```

??? success "Expected observations"
    The final successful request counter is `4`; the initialized `500` counter is `0`. The in-progress snapshot has gauge value `1`; the final snapshot has `0`. The Histogram and Summary each have count `4` and sum approximately `1.86` seconds. The `404` request series is absent.

    `# HELP` describes the instrument and `# TYPE` declares its exposition type. Numeric sample lines carry values. Comments do not each create a series. Label ordering and float formatting need not match the order you used in Python.

!!! tip "Zero and absent tell different stories"
    The `500` series exists with value zero because you initialized it. An absent `404` series does not by itself establish zero requests. Initialize the finite, valid combinations you need to distinguish. See [Python label initialization](https://prometheus.github.io/client_python/instrumenting/labels/).

Run the script again, saving a second final snapshot:

``` bash
.venv-metrics/bin/python experiments/metric_types.py > evidence/0003-types-second-run.prom
```

Predict whether the successful counter is now four or eight. It is **four**: each run creates a new process and registry, starting from zero before applying four increments. In a long-running application, successive requests update the same instrument. A process restart resets that state; future rate queries must handle resets. Python counters reject negative increments; use a Gauge for values that decrease.

## 4. Read cumulative buckets and compare Summary

For the final `/hello` Histogram snapshot, predict each bucket before expanding the answer. `le` means **less than or equal to** the boundary.

??? success "Expected cumulative bucket values"

    | `le` | Count | Observations included |
    | :--- | ---: | :--- |
    | `0.05` | 1 | `0.04` |
    | `0.1` | 1 | `0.04` |
    | `0.3` | 2 | `0.04`, `0.12` |
    | `0.5` | 3 | `0.04`, `0.12`, `0.5` |
    | `1.0` | 3 | The same three |
    | `2.5` | 4 | All four |
    | `+Inf` | 4 | All four |

    `_count` is `4`, matching the `+Inf` bucket. `_sum` is approximately `1.86` seconds. Do not add the bucket counts: an observation belongs to every bucket whose boundary includes it.

For these four observations, the cumulative mean is `1.86 / 4 = 0.465` seconds. Exactly two of four observations were at most `0.3` seconds. The boundary directly supports that threshold question. Estimating a percentile inside a bucket is a separate calculation covered in lesson 0010.

The Python Summary reports `_count` and `_sum`, plus optional `_created`; it does **not** provide `quantile="0.95"`. Other clients can implement Summary quantiles, but averaging instance percentiles does not produce a fleet percentile. Histograms support combining compatible bucket counts before estimating a percentile. See [Python Summary](https://prometheus.github.io/client_python/instrumenting/summary/) and [histogram/Summary tradeoffs](https://prometheus.io/docs/practices/histograms/).

## 5. Budget series before adding labels

**Cardinality** is the number of distinct series. Estimate the possible label combinations, then account for the sample names emitted by each instrument. A product of domain sizes is an upper bound; actual output depends on the combinations instantiated. A repeated request to an existing combination changes values without creating another series.

Use this capacity-planning example, distinct from the four-request experiment:

- Two permitted methods: `GET`, `POST`.
- Four route templates: `/hello`, `/items/{item_id}`, `/slow`, `/error`.
- Three status strings: `200`, `404`, `500`.
- Two application instances with the same schema.
- Six finite latency boundaries, plus the automatically included `+Inf`.

Some method/route/status combinations may be impossible in the eventual application. This calculation deliberately budgets the full product; initialize only valid combinations when implementing the API.

| Shared application metric | Label combinations per instance | Series per combination, excluding `_created` | Per instance, excluding `_created` | Per instance, including `_created` |
| :--- | :--- | :--- | ---: | ---: |
| Counter `app_http_requests_total` | `2 × 4 × 3 = 24` | 1 | 24 | 48 |
| Histogram `app_http_request_duration_seconds` | `2 × 4 = 8` | `7 buckets + sum + count = 9` | 72 | 80 |
| Gauge `app_http_requests_in_progress` | 1 | 1 | 1 | 1 |
| **Total** | | | **97** | **129** |
| **Two instances** | | | **194** | **258** |

The client exports one `_created` series per Counter, Histogram, or Summary child by default. A histogram child therefore emits ten series here, rather than nine, when `_created` is enabled. See [created-series behavior](https://prometheus.github.io/client_python/instrumenting/#disabling-created-metrics) and the [pinned client implementation](https://github.com/prometheus/client_python/blob/v0.26.0/prometheus_client/metrics.py).

These numbers budget exported application series before any scrape-time filtering. They exclude Prometheus-generated health series, default process metrics, and the comparison Summary. They are not an exact prediction of your TSDB's entire series count.

!!! note "Target labels and instrument labels"
    In future scrape configuration, Prometheus supplies `job="demo-api"` and `instance`, and you add `service="demo-api"` and `env="local"`. Keep `service` and `env` out of repeated metric definitions. One constant service/environment adds no independent multiplier here; two distinct instance label sets do. Budget the actual combinations rather than multiplying correlated labels twice.

Compare the first script without `_created`:

``` bash
.venv-metrics/bin/python experiments/metric_types.py --without-created > evidence/0003-types-without-created.prom
```

Expected: the same business values and histogram buckets, with no `_created` sample lines. For the small script's actual children, there are **18 exported series with `_created`** and **14 without**: two counter children, one histogram child, one unlabeled gauge, and one unlabeled Summary. Its counts differ from the capacity table because it initializes fewer combinations and includes the comparison Summary.

Disabling an optional series can reduce output, but it cannot make an unbounded label domain safe.

## 6. Failure and recovery: raw paths create new series

Suppose the router receives `/items/123` and `/items/456`. Both use the route template `/items/{item_id}`. Recording each raw path as a label creates one child per item identifier. Changing UUIDs, user IDs, request IDs, and timestamps produce similar growth. Use detailed logs/traces for individual events; keep metrics labels bounded. See [label cardinality guidance](https://prometheus.io/docs/practices/instrumentation/#use-labels).

Save `experiments/cardinality_lab.py`:

``` python
import argparse
import json

from prometheus_client import (
    CollectorRegistry,
    Counter,
    Histogram,
    disable_created_metrics,
    enable_created_metrics,
)

parser = argparse.ArgumentParser()
parser.add_argument("--raw-paths", action="store_true")
parser.add_argument("--without-created", action="store_true")
parser.add_argument("--items", type=int, default=100)
args = parser.parse_args()
if args.items < 1:
    parser.error("--items must be positive")
if args.without_created:
    disable_created_metrics()
else:
    enable_created_metrics()

registry = CollectorRegistry()
requests = Counter(
    "app_http_requests_total",
    "Completed application requests",
    ["method", "route", "status"],
    registry=registry,
)
duration = Histogram(
    "app_http_request_duration_seconds",
    "Application request duration in seconds",
    ["method", "route"],
    buckets=(0.05, 0.1, 0.3, 0.5, 1.0, 2.5),
    registry=registry,
)

for item_id in range(args.items):
    raw_path = f"/items/{item_id}"
    route = raw_path if args.raw_paths else "/items/{item_id}"  # (1)!
    requests.labels(method="GET", route=route, status="200").inc()
    duration.labels(method="GET", route=route).observe(0.12)

samples = [sample for family in registry.collect() for sample in family.samples]
counter_samples = [s for s in samples if s.name == "app_http_requests_total"]
print(json.dumps({
    "mode": "raw-paths" if args.raw_paths else "route-template",
    "completed_requests": sum(s.value for s in counter_samples),
    "counter_children": len(counter_samples),
    "created_series": sum(s.name.endswith("_created") for s in samples),
    "exported_series": len(samples),  # (2)!
}, indent=2))
```

1. This is a deliberately small simulation of one matched route, not a general path-normalization algorithm. Lesson 0004 obtains templates from the router and gives unmatched routes a bounded fallback.
2. Count all sample names and label sets in these two metric families. This script does not register the in-progress Gauge or the comparison Summary.

Predict the results for 100 distinct item IDs, then reproduce the failure and repair:

``` bash
.venv-metrics/bin/python experiments/cardinality_lab.py --raw-paths > evidence/0003-cardinality-raw.json
.venv-metrics/bin/python experiments/cardinality_lab.py > evidence/0003-cardinality-bounded.json
cat evidence/0003-cardinality-raw.json
cat evidence/0003-cardinality-bounded.json
```

??? success "Expected result and explanation"

    | Mode | Completed requests | Counter children | `_created` series | Exported series |
    | :--- | ---: | ---: | ---: | ---: |
    | Raw paths | 100 | 100 | 200 | 1,200 |
    | Route template | 100 | 1 | 2 | 12 |

    Each counter child emits two series and each histogram child ten. Raw paths create 100 combinations for both instruments: `100 × (2 + 10) = 1,200`. The bounded route creates one combination: `1 × (2 + 10) = 12`. The reduction is **100-fold**, while the request count remains 100.

    A new process/registry makes the comparison reproducible. Fixing label generation in a long-lived process does not erase its old metric children or historical Prometheus samples. Repair the instrumentation and manage the transition deliberately.

Run both modes again with `--without-created`. Predict the two exported-series counts and save the results:

``` bash
.venv-metrics/bin/python experiments/cardinality_lab.py --raw-paths --without-created > evidence/0003-cardinality-raw-without-created.json
.venv-metrics/bin/python experiments/cardinality_lab.py --without-created > evidence/0003-cardinality-bounded-without-created.json
```

Expected: **1,000 versus 10** exported series, with zero `_created` series. The raw-path growth remains. As a further prediction, try `--items 200`: the bounded schema should keep its series count while its request count doubles.

## 7. Independent design challenge

Create `evidence/0003-metric-design.md`. Use the shared application names and record:

1. The operational question, metric type, unit, and label domains for each of the three application metrics.
2. The six finite bucket boundaries, `+Inf`, and the distinction between instrument labels and scrape target labels.
3. The budget for one and two instances, both with and without `_created`; identify full-product assumptions and impossible combinations.
4. The observed raw/bounded counts and a label you rejected, explaining why its domain would grow.
5. How you would label a request to `/items/123`, an unmatched route, and a request that fails. Keep `/metrics` out of business request instrumentation.

**Challenge:** A proposed change adds the three-valued `status` label to the latency Histogram. With all other capacity-table assumptions unchanged and `_created` enabled, calculate the new total for one and two instances. Would it fit a hypothetical limit of 200 application series per instance? Explain whether status-specific latency is needed before approving the extra dimension.

??? success "Challenge calculation"
    Histogram combinations become `2 × 4 × 3 = 24`, each emitting ten series, for `240` histogram series per instance. Add `48` counter series and one gauge: **289 per instance**, or **578 across two instances**, exceeding the stated per-instance limit. The original shared contract keeps Histogram labels at `method,route`; the Counter already preserves status-specific request counts.

    A budget of 200 here is an exercise constraint, not a universal Prometheus limit. The useful decision is whether the added question justifies the extra series.

## 8. Active recall

Answer without reopening the scripts, then expand each answer.

??? question "Which instrument measures queue depth, and why?"
    A Gauge, because the number of queued items increases and decreases. A Counter would measure something accumulating, such as total enqueued items.

??? question "Why can the successful request counter show four in two separate script runs?"
    Each process creates fresh in-memory metric state, then increments four times. Repeated requests within one process would keep accumulating. A restart can reset a Counter.

??? question "What does the histogram bucket le=0.3 count?"
    Observations at most 0.3 seconds, including every smaller observation. Buckets overlap cumulatively, so adding bucket counts double-counts observations.

??? question "Does the Python Summary provide p95?"
    No. It provides count and sum, plus optional creation metadata. Use the course Histogram for percentile estimates in PromQL. Averaging precomputed instance percentiles from other clients would not yield a fleet percentile.

??? question "Why is request ID a poor metric label even if each request is short?"
    The number of distinct identifiers grows with traffic. Each new value creates a distinct series identity; finishing a request does not make that identity a bounded dimension.

??? question "Why do six finite histogram boundaries produce nine series before _created?"
    There are seven bucket series after including +Inf, plus one sum and one count. That is nine per instantiated non-bucket label combination.

??? question "Does zero traffic produce the same exposition as an absent label combination?"
    No. An initialized child can export zero. A combination never initialized or observed can have no series at all. Check the initialization policy before interpreting absence.

??? question "Why does two instances multiply the application budget while one constant env label does not?"
    Distinct instance label values separate each target's series. A constant environment label on those same series adds no further independent combinations.

## Verification and session handoff

- [ ] Explain the type and unit of each shared application metric.
- [ ] Run both Python files using the pinned environment and save their output.
- [ ] Find `HELP`, `TYPE`, the zero-valued initialized error series, and the absent `404` combination.
- [ ] Explain the gauge's `1` and `0` snapshots and the Counter's behavior across fresh processes.
- [ ] Reproduce the cumulative bucket table, sum, count, and Summary limitations.
- [ ] Reconcile the small script's 18/14 series with the separate capacity budget's 129/97 per instance.
- [ ] Demonstrate the raw-path failure and bounded-route recovery without changing the completed request total.
- [ ] Complete the design sheet and independently calculate the extra-label challenge.

**Author verification (2026-10-24):** Both scripts and the numeric results above were checked with Python `3.14.6` against the official client `v0.26.0` release source. Package-index requests timed out in the authoring environment, so installation from PyPI was not verified here.

**Files created during the exercise:** `monitoring-lab/experiments/requirements-metrics.txt`, `metric_types.py`, and `cardinality_lab.py`; `.venv-metrics/`; and evidence under `monitoring-lab/evidence/0003-*`. The Python filenames above both live under `monitoring-lab/experiments/`.

**Resume:** From `monitoring-lab/`, rerun the `.venv-metrics/bin/python` commands above. If the environment is missing, recreate it and reinstall from the requirements file. These scripts exit on their own and need no service stop command. If you ran the earlier Compose stack, `docker compose up -d` resumes it and `docker compose down` stops it while preserving named volumes.

**Next:** [Lesson 0004](0004-instrumenting-a-fastapi-application.md) turns these instruments into one FastAPI application with one Uvicorn worker, router-derived labels, monotonic request timing, exception cleanup, and `/metrics`. Keep the names, units, and chosen finite bucket boundaries from your design sheet. Prometheus target labels and application ingestion follow in lesson 0005.

Authored material and expected results do not establish learner completion. Use your evidence and recall responses to demonstrate understanding.

``` text
$teach Guide me through lesson 0004 in docs/lessons/0004-instrumenting-a-fastapi-application.md.
```

| Previous Lesson | All Lessons | Next Lesson |
| :--- | :---: | ---: |
| [0002: Configure Prometheus Scrape Jobs, Intervals, and Target Health](0002-prometheus-configuration-and-scraping.md) | [Lesson index](index.md) | [0004: Instrument a FastAPI Application and Expose /metrics](0004-instrumenting-a-fastapi-application.md) |
