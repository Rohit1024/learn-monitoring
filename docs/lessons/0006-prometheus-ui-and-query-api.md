---
icon: lucide/search
---

# 0006: Explore Metrics in the Prometheus UI and HTTP Query API

**Time:** 60–90 minutes. **Prerequisites:** [lesson 0005](0005-scraping-application-metrics.md), a terminal, `curl`, Python 3, and the shared container stack. Use Prometheus `v3.13.5`, Grafana `13.2.4`, and the unchanged, single-worker application from lesson 0004.

By the end, you should be able to:

- Investigate one series in Table and Graph, identifying its labels and evaluation times.
- Reproduce an investigation through instant and range HTTP queries.
- Distinguish scrape interval, evaluation time, and graph step.
- Recover from an empty result without mistaking it for a zero or a query error.
- Inspect target health, active configuration, metric discovery, and TSDB cardinality.

**Observable outcome:** A query notebook and saved JSON evidence connecting a controlled traffic burst to the same counter in the UI and API. This session teaches the investigation workflow; selectors, range vectors, rates, and histogram calculations follow in lessons 0007–0010.

## 1. Separate collection from evaluation

Prometheus already scrapes the app every fifteen seconds. Running a query reads stored data; it does not request a new scrape from FastAPI.

An **instant query** evaluates an expression at one timestamp. A **range query** evaluates it repeatedly between a start and end timestamp. The **step** is the spacing between those evaluations.

``` mermaid
flowchart TD
    Collection["App exposes metrics; Prometheus scrapes every 15 seconds"]
    Storage["Stored samples retain scrape timestamps"]
    Request["UI or API supplies expression and evaluation time(s)"]
    Engine["PromQL evaluates against available stored samples"]
    Kind{"One time or multiple times?"}
    Table["Table / instant API: one evaluation"]
    Graph["Graph / range API: evaluations separated by step"]
    Evidence["Inspect labels, values, and timestamps; record interpretation"]
    Collection --> Storage --> Engine
    Request --> Engine
    Engine --> Kind
    Kind -->|"One"| Table
    Kind -->|"Multiple"| Graph
    Table --> Evidence
    Graph --> Evidence
```

For a simple selector, an evaluation can use a recent sample taken before its timestamp. The default lookback is five minutes, subject to staleness. It does not mean that missing data is always carried forward for five minutes. See [querying basics](https://prometheus.io/docs/prometheus/latest/querying/basics/).

| Setting | Meaning | Today's value |
| :--- | :--- | :--- |
| Scrape interval | How often Prometheus collects from the app | `15s` |
| Instant evaluation time | When to evaluate an expression | An explicit Unix timestamp |
| Range start/end | Bounds of repeated evaluation | Saved around a traffic experiment |
| Range step | Spacing between evaluations | Compare `15s` and `5s` |

!!! note "A graph point is not necessarily a new measurement"
    A five-second graph step can reuse the same sample across evaluations when scrapes are fifteen seconds apart. It cannot recover an in-progress gauge spike that no scrape observed. A counter graph shows accumulated counts, not requests per second.

## 2. Resume a healthy lab

From the workspace root:

``` bash
mkdir -p monitoring-lab/evidence
cd monitoring-lab
docker version
docker compose version
python3 --version
```

**All remaining shell commands run from `monitoring-lab/`**, using the same terminal so saved variables remain available. Python is used only for standard-library JSON/time inspection; no new package installation is needed.

If files are missing, follow [lesson 0005, sections 2–3](0005-scraping-application-metrics.md#2-restore-the-starting-state): bootstrap the shared stack from lesson 0002, save the complete app files from lesson 0004, and add the app service and scrape configuration. The final configuration must scrape `app:8000/metrics` as `demo-api` with `service="demo-api"`, `env="local"`, and no deliberate metric filter.

If returning from lesson 0005's failure exercises, restore its known-good configuration first. Then:

``` bash
docker compose config --quiet
docker compose run --rm --no-deps --entrypoint /bin/promtool prometheus check config /etc/prometheus/prometheus.yml
docker compose up -d --build
curl --fail --silent --show-error http://localhost:9090/-/ready
curl --fail --silent --show-error --request POST http://localhost:9090/-/reload
curl --fail --silent --show-error --get http://localhost:9090/api/v1/query \
  --data-urlencode 'query=up{job="demo-api"}'
```

Repeat readiness if startup is still underway. Allow a fresh scrape before checking `up`; continue only when its result has value `"1"`. For persistent problems inspect `docker compose logs --tail=60 app prometheus` and use lesson 0005's diagnostics.

Capture the running version and target state:

``` bash
curl --fail --silent --show-error --output evidence/0006-buildinfo.json http://localhost:9090/api/v1/status/buildinfo
curl --fail --silent --show-error --output evidence/0006-targets.json 'http://localhost:9090/api/v1/targets?state=active'
```

Check `data.version` is `3.13.5`. In the `demo-api` target, inspect `scrapeUrl`, `labels`, `health`, `lastScrape`, and `lastError`.

## 3. Start in Table, then use Graph

Open `http://localhost:9090/query`. The pinned [v3.13.5 UI routes](https://github.com/prometheus/prometheus/blob/v3.13.5/web/ui/mantine-ui/src/App.tsx) use `/query`; the general [expression-browser guide](https://prometheus.io/docs/visualization/browser/) still describes `/graph`. Use the query page of your running version rather than assuming all documentation screenshots match.

1. Enter `up{job="demo-api"}` and execute it in **Table**. Identify the single result's `job`, `instance`, `service`, and `env`. A healthy result has value one.
2. Replace the expression with the counter below. Read every label before interpreting its number.
3. Select **Graph** and a five-minute range ending now. Look for a cumulative line: flat segments mean no observed counter increase, while a drop can indicate a process reset.
4. Change the displayed range to fifteen minutes, then back to five. A newly started target cannot supply history before it was first scraped.
5. Locate the end-time and resolution controls. Use `15s` resolution, then `5s`; leave automatic refresh off when comparing fixed observations. If your version exposes different controls, use the API experiment below to set precise parameters.

``` promql
app_http_requests_total{job="demo-api",method="GET",route="/hello",status="200"}
```

Expected with this single-target lab: one series. Zero is a valid initialized counter value. More than one series means the complete label sets differ—inspect them rather than assuming the UI duplicated a result.

**Prediction:** If you change Graph resolution from fifteen to five seconds without sending traffic, should the counter increase? Explain before continuing.

??? success "Prediction check"
    No. Query resolution does not generate business traffic or scrape samples. More evaluated points can repeat the same value.

## 4. Capture a fixed instant query

Define the expression and timestamp, and save both for reproducibility:

``` bash
lesson_query='app_http_requests_total{job="demo-api",method="GET",route="/hello",status="200"}'
lesson_start=$(date +%s)
printf '%s\n' "$lesson_query" > evidence/0006-query.promql
printf '%s\n' "$lesson_start" > evidence/0006-start.txt
curl --fail --silent --show-error --get http://localhost:9090/api/v1/query \
  --data-urlencode "query=$lesson_query" \
  --data-urlencode "time=$lesson_start" \
  --output evidence/0006-instant-before.json
python3 -m json.tool evidence/0006-instant-before.json
```

`--get` sends query parameters on a GET request; `--data-urlencode` preserves braces, quotes, and other PromQL characters. Quoting the shell variable prevents word splitting. Explicit `time` fixes the evaluation instant; omitting it uses the server's current time. See the [pinned HTTP API specification](https://github.com/prometheus/prometheus/blob/v3.13.5/docs/querying/api.md).

Read the response in this order:

| Field | What to check |
| :--- | :--- |
| `status` | `success` means the API query succeeded; it does not assert app health |
| `data.resultType` | This selector returns `vector` |
| `data.result` | Array of selected series; expect one here |
| `metric` | Full label set identifying the series |
| `value` | Pair `[evaluation_timestamp, "numeric_value"]` |

Values are encoded as strings in this JSON format. The timestamp of an instant-vector result is the evaluation time; it is not proof of a scrape at that exact instant.

To inspect the timestamp of the underlying sample selected by this plain selector:

``` bash
curl --fail --silent --show-error --get http://localhost:9090/api/v1/query \
  --data-urlencode "query=timestamp($lesson_query)" \
  --data-urlencode "time=$lesson_start" \
  --output evidence/0006-source-timestamp.json
python3 -m json.tool evidence/0006-source-timestamp.json
```

In this result, the **numeric value** is the selected sample's Unix timestamp in seconds, while the surrounding pair still uses the evaluation timestamp. Normally the sample time is slightly earlier. See [`timestamp()`](https://prometheus.io/docs/prometheus/latest/querying/functions/#timestamp).

## 5. Create a visible traffic burst

Save a baseline as above before sending traffic. Keep the same app process running, and avoid other requests to `/hello` during this exercise.

``` bash
for request_number in 1 2 3 4 5 6; do
  curl --fail --silent --show-error --output /dev/null http://localhost:8000/hello
  sleep 5
done
sleep 20
lesson_end=$(date +%s)
printf '%s\n' "$lesson_end" > evidence/0006-end.txt
curl --fail --silent --show-error --get http://localhost:9090/api/v1/query \
  --data-urlencode "query=$lesson_query" \
  --data-urlencode "time=$lesson_end" \
  --output evidence/0006-instant-after.json
curl --fail --silent --show-error --output evidence/0006-targets-after.json 'http://localhost:9090/api/v1/targets?state=active'
```

The final pause allows collection after the burst; inspect target health and `lastScrape` to confirm that it actually occurred. Predict the counter difference, then calculate it:

``` bash
python3 - <<'PY'
import json
from pathlib import Path

def one_value(filename):
    payload = json.loads(Path(filename).read_text())
    assert payload["status"] == "success", payload
    assert payload["data"]["resultType"] == "vector", payload
    rows = payload["data"]["result"]
    assert len(rows) == 1, f"Expected one series, got {len(rows)}"
    return rows[0]["metric"], float(rows[0]["value"][1])

before_labels, before = one_value("evidence/0006-instant-before.json")
after_labels, after = one_value("evidence/0006-instant-after.json")
assert before_labels == after_labels, "Series identity changed"
print({"before": before, "after": after, "delta": after - before})
PY
```

Expected delta: **six**, provided the baseline included earlier requests, all six requests succeeded, a subsequent scrape completed, and the app did not restart. If it differs, investigate those conditions; do not edit the saved evidence to fit the prediction.

Return to Graph with an end time matching `lesson_end` and a range covering `lesson_start`. Locate the increase. Convert the timestamps for entering a fixed UTC time in the UI:

``` bash
python3 - <<'PY'
from datetime import datetime, timezone
from pathlib import Path
for name in ("start", "end"):
    value = int(Path(f"evidence/0006-{name}.txt").read_text())
    print(name, datetime.fromtimestamp(value, timezone.utc).isoformat())
PY
```

Account for the UI's displayed timezone. For an exact Table/API comparison, set Table's evaluation time to the saved end time and execute the same expression. Comparing two independent “now” queries can differ simply because another scrape occurred between them.

## 6. Repeat the range through the API

Load saved parameters so these commands also work after opening a new terminal in `monitoring-lab/`:

``` bash
lesson_query=$(cat evidence/0006-query.promql)
lesson_start=$(cat evidence/0006-start.txt)
lesson_end=$(cat evidence/0006-end.txt)
for lesson_step in 15s 5s; do
  curl --fail --silent --show-error --get http://localhost:9090/api/v1/query_range \
    --data-urlencode "query=$lesson_query" \
    --data-urlencode "start=$lesson_start" \
    --data-urlencode "end=$lesson_end" \
    --data-urlencode "step=$lesson_step" \
    --output "evidence/0006-range-$lesson_step.json"
done
python3 -m json.tool evidence/0006-range-15s.json
```

A range response has `resultType: "matrix"`. Each series has a `metric` label object and a `values` array of timestamp/value pairs, instead of the instant response's single `value` pair. Start and end bounds are inclusive; evaluations occur at `start + n × step` while that time is no later than `end`. An unaligned end need not appear as a point.

Count and inspect the evaluations:

``` bash
python3 - <<'PY'
import json
from pathlib import Path

start = int(Path("evidence/0006-start.txt").read_text())
end = int(Path("evidence/0006-end.txt").read_text())
for step in (15, 5):
    payload = json.loads(Path(f"evidence/0006-range-{step}s.json").read_text())
    assert payload["status"] == "success", payload
    assert payload["data"]["resultType"] == "matrix", payload
    rows = payload["data"]["result"]
    assert len(rows) == 1, f"Expected one series, got {len(rows)}"
    points = rows[0]["values"]  # (1)!
    print({"step_seconds": step,
           "scheduled_evaluations": (end - start) // step + 1,
           "returned_points": len(points),
           "first": points[0], "last": points[-1]})
PY
```

1. These are query evaluation points. Do not count them as newly ingested samples or unique requests.

Expect more returned points at five seconds, often with repeated values. If the series was present throughout, the point count matches the scheduled count. Staleness or absence can leave evaluations without a returned sample.

!!! tip "Keep the end-point comparison honest"
    The final range point can precede `lesson_end`. Compare it with an instant query at that point's timestamp, or choose an aligned range. A mismatch with an instant query at a later time is not necessarily an API inconsistency.

A range-query API response called `matrix` is not a command to add `[5m]` to the expression. That bracket syntax selects a window inside PromQL and is taught in lesson 0007. Here the same plain selector is evaluated repeatedly by `/query_range`.

## 7. Discover metrics and inspect collection state

Use the query editor's metric-name suggestions to find `app_http_requests_total`. Then use these reproducible API equivalents to inspect metadata:

``` bash
curl --fail --silent --show-error --get http://localhost:9090/api/v1/label/__name__/values \
  --data-urlencode 'match[]={job="demo-api"}' \
  --data-urlencode "start=$lesson_start" \
  --data-urlencode "end=$lesson_end" \
  --output evidence/0006-metric-names.json
curl --fail --silent --show-error --get http://localhost:9090/api/v1/series \
  --data-urlencode "match[]=$lesson_query" \
  --data-urlencode "start=$lesson_start" \
  --data-urlencode "end=$lesson_end" \
  --output evidence/0006-series.json
curl --fail --silent --show-error --output evidence/0006-active-config.json http://localhost:9090/api/v1/status/config
curl --fail --silent --show-error --output evidence/0006-tsdb.json 'http://localhost:9090/api/v1/status/tsdb?limit=10'
```

Inspect the files with `python3 -m json.tool <filename>`. In the UI, find the pages for targets, configuration, and TSDB status; use the saved API output if menu placement differs.

- **Discovery:** Metric names and label sets help construct a query. Metadata endpoints can return series metadata even without a sample at your precise evaluation time; discovery is not a substitute for a query.
- **Targets:** Confirm the current address, health, labels, last scrape, and error for `demo-api`.
- **Active configuration:** Confirm the running server loaded the intended job. This is stronger evidence than reading only a local YAML file.
- **TSDB:** Inspect `headStats.numSeries`, `seriesCountByMetricName`, and `labelValueCountByLabelName`. These statistics describe the database's current head, across jobs, not just the app or the historical graph window. Head timestamps are milliseconds; query timestamps above are seconds.

The TSDB's top-ten lists may omit your app metrics. Its series count is not a request count, and should not equal the metric-name count: labels and histogram buckets create multiple series. Repeated bounded `/hello` traffic changes values without requiring new label combinations. See the [v3.13.5 API metadata and TSDB definitions](https://github.com/prometheus/prometheus/blob/v3.13.5/docs/querying/api.md).

## 8. Failure and recovery: empty results versus errors

### A. A valid expression with the wrong job

``` bash
curl --silent --show-error --get http://localhost:9090/api/v1/query \
  --data-urlencode 'query=up{job="demo-api-typo"}' \
  --data-urlencode "time=$lesson_end" \
  --output evidence/0006-wrong-job.json --write-out '%{http_code}\n'
```

Expected: HTTP 200, `status: "success"`, and an empty `result` array. No series matches that label. Repair the job to `demo-api` and repeat at the same time. An empty array is not a series with value zero.

### B. Correct selector at a time without history

Use the deliberately old Unix timestamp `1` (1970-01-01T00:00:01Z), before this lab existed:

``` bash
curl --fail --silent --show-error --get http://localhost:9090/api/v1/query \
  --data-urlencode "query=$lesson_query" \
  --data-urlencode 'time=1' \
  --output evidence/0006-wrong-time.json
```

Expected: successful empty result. Recover by using `time=$lesson_end`. For a graph, restore a range overlapping the saved traffic experiment. A healthy target today cannot supply data from before it was collected.

### C. An invalid expression

``` bash
curl --silent --show-error --get http://localhost:9090/api/v1/query \
  --data-urlencode 'query=up{job="demo-api"' \
  --output evidence/0006-parse-error.json --write-out '%{http_code}\n'
python3 -m json.tool evidence/0006-parse-error.json
```

Expected: HTTP 400 and a JSON error describing the missing closing brace. `--fail` is deliberately omitted here so you can save the error body. Repair the expression to `up{job="demo-api"}` and rerun. If the connection itself fails, there may be no Prometheus JSON response at all; check service readiness first.

``` mermaid
flowchart TD
    Result["Unexpected query result"]
    Response{"Did the API return success?"}
    Error["No: inspect transport, HTTP status, and error fields"]
    Empty{"Is result empty?"}
    Labels["Yes: verify metric, labels, and evaluation time"]
    Values["No: inspect full label sets, values, and timestamps"]
    Collection["Check target health and collection history if still unexplained"]
    Repair["Repair one cause and rerun with saved parameters"]
    Result --> Response
    Response -->|"No"| Error
    Response -->|"Yes"| Empty
    Empty -->|"Yes"| Labels
    Empty -->|"No"| Values
    Labels --> Collection
    Values --> Collection
    Error --> Repair
    Collection --> Repair
```

## 9. Independent investigation and notebook

Investigate this question without copying the `/hello` result: **Did requests to two different item IDs contribute to the same route-template counter?**

1. Select `app_http_requests_total` for GET, route `/items/{item_id}`, status `200`, and job `demo-api`. Capture a baseline at an explicit timestamp after a healthy scrape.
2. Send one request each to `/items/101` and `/items/202`. Allow a subsequent scrape, then save the final instant response.
3. Show that both contributed to the same series and explain the delta. Build a range request covering the experiment with a chosen step.
4. Reproduce the query in Table at the same evaluation time. Explain any UI/API discrepancy using evidence.
5. Deliberately change the route matcher to `/items/101`, predict the result, and recover using the template. No source change is needed.

Create `evidence/0006-query-notebook.md` using this template for the guided lab and independent investigation:

``` markdown
## Question
What am I trying to establish?

## Reproduction
- Expression:
- Instant evaluation time (Unix seconds and UTC):
- Range start/end/step, if used:
- Prometheus version and relevant target labels:
- Evidence filenames:

## Prediction and observations
- Expected series count and change:
- Actual labels, values, and result type:
- UI/API comparison at matching times:
- Empty/error experiment and repair:

## Interpretation
What does this establish? What cannot be inferred from these samples?
```

## 10. Active recall

??? question "What does an instant query evaluate?"
    An expression at one evaluation timestamp. It can select an earlier stored sample; it does not force a new scrape.

??? question "Does changing graph step from 15s to 5s triple stored samples?"
    No. It schedules more evaluations of stored data. Repeated values can come from the same selected sample.

??? question "How does a parse error differ from a successful empty result?"
    A parse error rejects the expression. A successful empty result evaluates valid syntax but returns no matching data at the requested time. Neither is the same as a returned value of zero.

??? question "Why save evaluation times with the expression?"
    The same expression evaluated later can select different samples. Fixed parameters make the investigation reproducible while the underlying history remains retained.

??? question "Why can the final range point differ from an instant query at end?"
    The step grid might not land exactly on end. Compare matching evaluation timestamps before concluding the values disagree.

??? question "Does discovering a metric name prove it has a sample now?"
    No. Metadata discovery and time-specific query evaluation answer different questions.

??? question "Does TSDB numSeries tell you how many requests the app handled?"
    No. It counts distinct series in the database head, including other jobs. Request totals are values within particular application counter series.

## Verification and session handoff

- [ ] Resume the healthy pinned stack and save version/target evidence.
- [ ] Inspect labels in Table and cumulative behavior in Graph.
- [ ] Capture explicit instant-query times and distinguish evaluation from source timestamps.
- [ ] Generate six requests and explain the observed counter delta.
- [ ] Compare fifteen- and five-second range steps and inspect vector/matrix JSON shapes.
- [ ] Inspect discovery, active configuration, and TSDB statistics without confusing them with current query results.
- [ ] Recover from wrong labels, absent historical data, and a parse error.
- [ ] Complete the item-route challenge and query notebook.

**Files created during the exercise:** `monitoring-lab/evidence/0006-*`. No application or scrape-configuration change is required after restoring the lesson 0005 baseline. Keep the same dependency pins and named volumes.

**Resume:** From `monitoring-lab/`, run `docker compose up -d --build`, check readiness and `up{job="demo-api"}`, and reload the saved query/start/end variables from section 6. Old evidence can be queried while its history remains retained; capture a new interval for new traffic. **Stop:** `docker compose down` preserves named volumes. App counters reset when its process restarts.

**Author verification:** Official query documentation and the pinned v3.13.5 UI/API source were checked. The local container engine was unavailable, so live UI/API observations are expected learner outcomes, not claimed executions. Documentation build and Mermaid validation results accompany the authoring handoff.

**Next:** [Lesson 0007](0007-promql-selectors-and-vectors.md) explores label matchers, instant/range vectors, lookback, and time windows. Authorship does not establish learner completion; retain your evidence and recall answers.

``` text
$teach Expand lesson 0007 using the outline in docs/lessons/0007-promql-selectors-and-vectors.md and docs/lesson-authoring.md.
```

| Previous Lesson | All Lessons | Next Lesson |
| :--- | :---: | ---: |
| [0005: Scrape Application Metrics and Troubleshoot Prometheus Targets](0005-scraping-application-metrics.md) | [Lesson index](index.md) | [0007: Select and Filter Time Series with PromQL Vectors and Time Windows](0007-promql-selectors-and-vectors.md) |
