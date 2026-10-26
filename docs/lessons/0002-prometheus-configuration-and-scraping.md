---
icon: lucide/radar
---

# 0002: Configure Prometheus Scrape Jobs, Intervals, and Target Health

**Time:** 60–90 minutes. **Prerequisites:** [lesson 0001](0001-setting-up-prometheus-and-grafana.md), a terminal, `curl`, and a running container engine with Docker Compose support. Keep the course pins: Prometheus `v3.13.5` and Grafana `13.2.4`.

By the end, you should be able to:

- Configure a job, its target addresses, labels, endpoint, interval, and timeout.
- Validate a file, reload Prometheus, and prove which configuration is active.
- Investigate scrape health using target metadata and three generated metrics.
- Distinguish a failed scrape, a removed target, and application errors.

**Observable outcome:** Save evidence of a healthy self-scrape, a deliberately unreachable target, a rejected configuration, and a restored baseline. The application arrives in lesson 0004; Prometheus itself supplies today's working metrics endpoint.

## 1. Understand what a scrape job controls

Recall the pipeline: a process exposes `/metrics`, Prometheus collects and stores samples, and Grafana queries that history. Today we control the collection step.

A **target** is an endpoint Prometheus scrapes. An **instance** identifies an individual target; a **job** groups instances with the same purpose. In this lab, `job_name: prometheus` supplies `job="prometheus"`, and the address supplies `instance="localhost:9090"`. See [jobs and instances](https://prometheus.io/docs/concepts/jobs_instances/).

``` mermaid
flowchart TD
    Disk["Edit prometheus/prometheus.yml on disk"]
    Validate["promtool checks the candidate configuration"]
    Reload["Reload the running Prometheus process"]
    Active["Inspect the active configuration"]
    Schedule["Each job schedules its target scrapes"]
    Request["HTTP request to the configured metrics path"]
    Result["Store scrape health and available metric samples"]
    Inspect["Inspect targets and query up, duration, and sample count"]
    Disk --> Validate
    Validate --> Reload
    Reload --> Active
    Active --> Schedule
    Schedule --> Request
    Request --> Result
    Result --> Inspect
```

!!! tip "Three separate checks"
    A valid file can point at an unreachable endpoint. A file on disk can differ from the running configuration. A successful scrape can coexist with application errors. Collect evidence for each layer.

## 2. Resume the shared lab

From the workspace root, check the engine and enter the lab:

``` bash
docker version
docker compose version
mkdir -p monitoring-lab/prometheus monitoring-lab/evidence
cd monitoring-lab
```

`docker version` must show a reachable server. If Docker is a Podman wrapper, start its existing VM and check the Compose provider as described in lesson 0001 before continuing.

**All remaining shell commands run from `monitoring-lab/`.** Keep the same directory and Compose project name to reuse the lesson 0001 volumes. If you changed host port 9090 in that lesson, adjust the host `curl` and browser URLs here; the self-scrape address still uses container port 9090.

If the files already exist, keep them and back up the configuration before editing:

``` bash
cp prometheus/prometheus.yml evidence/0002-before.yml
```

??? example "Starting without the lesson 0001 files"
    Create `docker-compose.yml` with the complete shared stack below. This restores the runnable starting state; review lesson 0001 if container networking or persistence is unfamiliar.

    ``` yaml
    services:
      prometheus:
        image: prom/prometheus:v3.13.5
        restart: unless-stopped
        ports:
          - "127.0.0.1:9090:9090"
        volumes:
          - ./prometheus:/etc/prometheus:ro
          - prometheus_data:/prometheus
        command:
          - --config.file=/etc/prometheus/prometheus.yml
          - --storage.tsdb.path=/prometheus
          - --web.enable-lifecycle
        networks:
          - monitoring

      grafana:
        image: grafana/grafana:13.2.4
        restart: unless-stopped
        ports:
          - "127.0.0.1:3000:3000"
        environment:
          GF_SECURITY_ADMIN_USER: admin
          GF_SECURITY_ADMIN_PASSWORD: admin
          GF_USERS_ALLOW_SIGN_UP: "false"
        volumes:
          - grafana_data:/var/lib/grafana
        networks:
          - monitoring
        depends_on:
          - prometheus

    networks:
      monitoring:
        driver: bridge

    volumes:
      prometheus_data: {}
      grafana_data: {}
    ```

    Create `prometheus/prometheus.yml` with the baseline:

    ``` yaml
    global:
      scrape_interval: 15s
      scrape_timeout: 10s
      evaluation_interval: 15s

    scrape_configs:
      - job_name: prometheus
        metrics_path: /metrics
        static_configs:
          - targets: ["localhost:9090"]
    ```

    Save the starting copy:

    ``` bash
    cp prometheus/prometheus.yml evidence/0002-before.yml
    ```

Validate and start the existing or restored stack:

``` bash
docker compose config --quiet
docker compose run --rm --no-deps --entrypoint /bin/promtool prometheus check config /etc/prometheus/prometheus.yml
docker compose up -d
docker compose ps
curl --fail --silent --show-error http://localhost:9090/-/ready
```

Expected: configuration validation succeeds, the services start, and readiness returns HTTP 200. Repeat readiness after a short startup delay if needed. Inspect `docker compose logs --tail=60 prometheus` if it continues to fail. The named volumes hold history separately from the read-only configuration directory.

## 3. Set explicit defaults and a job override

Replace `prometheus/prometheus.yml` with this complete exercise configuration:

``` yaml
global:
  scrape_interval: 15s  # (1)!
  scrape_timeout: 10s
  evaluation_interval: 30s  # (2)!

scrape_configs:
  - job_name: prometheus  # (3)!
    scrape_interval: 5s  # (4)!
    scrape_timeout: 3s
    metrics_path: /metrics  # (5)!
    static_configs:
      - targets: ["localhost:9090"]  # (6)!
        labels:
          service: prometheus
          env: local  # (7)!
```

1. Global settings supply defaults for jobs. This lab writes both interval and timeout explicitly.
2. This is the default recording/alerting rule evaluation schedule. There are no rule files yet.
3. The job name becomes the `job` label in this configuration.
4. This job uses a five-second interval and a three-second timeout instead of the global settings.
5. Prometheus requests this path; `/metrics` is also the default. The default scheme is HTTP.
6. Targets contain `host:port`, without a scheme or path. Here `localhost` refers to the Prometheus container itself.
7. These target labels accompany this target's stored samples. Future application jobs use `service: demo-api` instead.

Predict the scrape URL, resulting labels, and effective timing before reloading:

??? success "Expected configuration"
    The URL is `http://localhost:9090/metrics`. Its labels include `job="prometheus"`, `instance="localhost:9090"`, `service="prometheus"`, and `env="local"`. The job uses `5s` / `3s`; rule evaluation defaults to `30s`.

The five-second interval speeds up the exercises. We restore the course's fifteen-second baseline at the end. For roughly unchanged payloads, changing from fifteen to five seconds means about three times as many scrape requests and collected samples per minute.

| Setting | What it controls | Exercise value |
| :--- | :--- | :--- |
| `scrape_interval` | How often a target is scraped | `5s` for this job |
| `scrape_timeout` | Maximum duration allowed for one scrape | `3s` for this job |
| `evaluation_interval` | Default frequency for recording/alerting rule evaluation | `30s` |

The timeout must be no greater than the interval. Set both explicitly when experimenting. Changing the evaluation schedule does not change collection frequency. These fields are documented in the [configuration reference](https://prometheus.io/docs/prometheus/latest/configuration/configuration/).

!!! note "Labels change series identity"
    Adding `service` and `env` creates new series identities. Earlier self-scrape history still has its old label set. Use the selectors below for the new series; allow a scrape to complete after reloading.

## 4. Validate, reload, and inspect the active configuration

Check the candidate with the same pinned image that runs Prometheus:

``` bash
docker compose run --rm --no-deps --entrypoint /bin/promtool prometheus check config /etc/prometheus/prometheus.yml
```

Expected: exit status 0 and a success message. This checks configuration; it does not contact every target to prove reachability. If validation fails, repair the file before continuing.

Trigger one reload method:

=== "HTTP reload"

    Lesson 0001 enables the lifecycle endpoint with `--web.enable-lifecycle`:

    ``` bash
    curl --fail --silent --show-error --request POST http://localhost:9090/-/reload
    ```

    Expected: HTTP 200, often with an empty body. A GET request is not the reload operation. If lifecycle support is disabled, use the signal method or add the flag and recreate the service with `docker compose up -d prometheus`.

=== "Signal reload"

    Send `SIGHUP` to the running service:

    ``` bash
    docker compose kill --signal SIGHUP prometheus
    ```

    For the course image, this signals Prometheus to reload. Inspect logs and active configuration to verify the result.

Both methods are defined by the [management API](https://prometheus.io/docs/prometheus/latest/management_api/).

Read back the running configuration and reload status:

``` bash
curl --fail --silent --show-error http://localhost:9090/api/v1/status/config
curl --fail --silent --show-error http://localhost:9090/api/v1/status/runtimeinfo
curl --fail --silent --show-error --output evidence/0002-healthy-targets.json 'http://localhost:9090/api/v1/targets?state=active'
cat evidence/0002-healthy-targets.json
```

In `data.yaml`, find the job's `scrape_interval: 5s` and `scrape_timeout: 3s`; the JSON encodes line breaks as `\n`. Comments are omitted. In runtime information, check `reloadConfigSuccess` and `lastConfigTime`. The target response shows `scrapeUrl`, `scrapeInterval`, `scrapeTimeout`, `health`, `lastScrape`, and `lastError`. These endpoints are documented in the [HTTP API](https://prometheus.io/docs/prometheus/latest/querying/api/).

Expected after a completed scrape: the self-target is `up`, with an empty `lastError`, interval `5s`, and timeout `3s`. A newly added target may briefly be `unknown` before its first attempt. Use the target list in the UI if it is easier to read; API commands keep the exercise independent of menu names.

## 5. Query the scrape evidence

Open [Prometheus](http://localhost:9090) and run these separately in the expression browser's **Table** view:

``` promql
up{job="prometheus",service="prometheus",env="local"}
```

``` promql
scrape_duration_seconds{job="prometheus",service="prometheus",env="local"}
```

``` promql
scrape_samples_scraped{job="prometheus",service="prometheus",env="local"}
```

Predict each value's meaning before reading the table:

| Metric | Expected for the self-target | Meaning |
| :--- | :--- | :--- |
| `up` | `1` after success | Whether the latest scrape succeeded |
| `scrape_duration_seconds` | A nonnegative duration in seconds | How long the scrape took |
| `scrape_samples_scraped` | A positive count | Samples exposed in that scrape |

Prometheus generates these series for each scrape. They measure collection, not business request volume. `scrape_samples_scraped` is a per-scrape count, not a cumulative request counter. See [generated scrape series](https://prometheus.io/docs/concepts/jobs_instances/#automatically-generated-labels-and-time-series).

Save the healthy query response:

``` bash
curl --fail --silent --show-error --get http://localhost:9090/api/v1/query --data-urlencode 'query=up{job="prometheus",service="prometheus",env="local"}' --output evidence/0002-healthy-up.json
cat evidence/0002-healthy-up.json
```

Expected: `status: success`, a vector result with the target labels, and a value string `"1"`. Record the actual sample count and duration in `evidence/0002-notes.md`; their exact values vary.

!!! tip "Scrape success and application success"
    Imagine a service whose `/metrics` works while `/hello` returns HTTP 500. Its scrape can still succeed. Later lessons add request and status metrics so you can investigate business responses separately.

## 6. Add an unreachable target and diagnose it

Append this second job **inside the existing `scrape_configs` list**, aligned with the first `- job_name`. Keep the working self-scrape job:

``` yaml
  - job_name: unreachable-demo
    scrape_interval: 5s
    scrape_timeout: 2s
    metrics_path: /metrics
    static_configs:
      - targets: ["127.0.0.1:65535"]  # (1)!
        labels:
          env: local
```

1. The course Prometheus container has no listener on this loopback port. This creates a local connection failure without depending on an external host or DNS service.

Before running it, predict: will `promtool` accept the file? Which job will fail? Will readiness fail too?

``` bash
docker compose run --rm --no-deps --entrypoint /bin/promtool prometheus check config /etc/prometheus/prometheus.yml
curl --fail --silent --show-error --request POST http://localhost:9090/-/reload
```

After at least one five-second scrape cycle, inspect the targets and save the failed query:

``` bash
curl --fail --silent --show-error --output evidence/0002-unreachable-targets.json 'http://localhost:9090/api/v1/targets?state=active'
cat evidence/0002-unreachable-targets.json
curl --fail --silent --show-error --get http://localhost:9090/api/v1/query --data-urlencode 'query=up{job="unreachable-demo"}' --output evidence/0002-unreachable-up.json
cat evidence/0002-unreachable-up.json
curl --fail --silent --show-error http://localhost:9090/-/ready
```

Expected observations:

- Validation and reload succeed: the address is valid configuration.
- The self-target stays healthy; the new target reports `health: down` and a connection-refused `lastError`.
- `up{job="unreachable-demo"}` returns `0`; readiness still succeeds.
- The failed target's scrape duration can be very short. A refusal need not consume its full timeout.

Query its duration and sample count too. For this refused connection, expect zero scraped samples; use `lastError` to diagnose the cause. A duration alone cannot distinguish a fast success from a fast failure.

``` mermaid
flowchart TD
    Start["Expected target is unhealthy or missing"]
    Listed{"Listed in active targets?"}
    Missing["Check active configuration, job spelling, and reload result"]
    Health{"Completed scrape succeeded?"}
    Failed["Read lastError and the exact scrapeUrl"]
    Refused["For this lab: no listener at 127.0.0.1:65535"]
    Success["Investigate expected application metrics separately"]
    Start --> Listed
    Listed -->|"No"| Missing
    Listed -->|"Yes"| Health
    Health -->|"No"| Failed
    Failed --> Refused
    Health -->|"Yes"| Success
```

Keep the failed job for the next exercise. It provides a useful marker when comparing disk and active configuration.

## 7. Independent challenge: reject an invalid timeout and recover

Save the valid two-job configuration:

``` bash
cp prometheus/prometheus.yml evidence/0002-working.yml
```

In the **`prometheus` job**, leave `scrape_interval: 5s` and change its explicit `scrape_timeout` from `3s` to `6s`. Leave the global timeout alone.

Try these tasks before expanding the answer:

1. Run `promtool check config` using the command from section 4. Identify the job and constraint in its error.
2. Predict whether the running process has already changed. Compare `/api/v1/status/config` with the edited file.
3. For this controlled failure only, send the invalid candidate to the reload endpoint. Check readiness and active configuration again.
4. Restore `evidence/0002-working.yml`, validate, reload, and prove recovery.

??? success "Expected failure and recovery"
    Validation exits nonzero with an error containing `scrape timeout greater than scrape interval` and the job name `prometheus`. The explicit six-second timeout exceeds the five-second interval. This validation behavior is present in the [pinned v3.13.5 configuration implementation](https://github.com/prometheus/prometheus/blob/v3.13.5/config/config.go).

    Editing the file does not activate it in this lab. Deliberately attempt the reload:

    ``` bash
    curl --fail --silent --show-error --request POST http://localhost:9090/-/reload
    curl --fail --silent --show-error http://localhost:9090/api/v1/status/config
    curl --fail --silent --show-error http://localhost:9090/api/v1/status/runtimeinfo
    curl --fail --silent --show-error http://localhost:9090/-/ready
    docker compose logs --tail=40 prometheus
    ```

    Expected: the reload returns HTTP 500, so `curl --fail` exits nonzero. The old active configuration still shows `3s`, both old jobs remain, and readiness succeeds. Runtime information reports the unsuccessful reload. Prometheus preserves the previously loaded configuration when rejecting an invalid candidate; see [configuration reload behavior](https://prometheus.io/docs/prometheus/latest/configuration/configuration/).

    Restore a valid file on disk before any restart:

    ``` bash
    cp evidence/0002-working.yml prometheus/prometheus.yml
    docker compose run --rm --no-deps --entrypoint /bin/promtool prometheus check config /etc/prometheus/prometheus.yml
    curl --fail --silent --show-error --request POST http://localhost:9090/-/reload
    curl --fail --silent --show-error --output evidence/0002-recovered-runtime.json http://localhost:9090/api/v1/status/runtimeinfo
    cat evidence/0002-recovered-runtime.json
    ```

    Expected: validation succeeds, reload returns 200, and `reloadConfigSuccess` returns to `true`. The intentionally unreachable target remains down until you remove it in section 8.

!!! note "A rejected reload is different from a failed startup"
    The running process has a previous configuration to keep. A fresh process starting with the invalid file cannot rely on that in-memory state. Repair the disk file before stopping or recreating the service.

In `evidence/0002-notes.md`, explain what failed, what kept working, and how you proved the repaired configuration was loaded. Include the validation error and the relevant runtime response.

## 8. Remove the failed target and restore the course baseline

Predict what happens to `up{job="unreachable-demo"}` after removal. Will Prometheus keep writing zero, turn it into one, or stop producing it?

First remove the target while retaining its job. Replace only the `unreachable-demo` job block with this empty target list:

``` yaml
  - job_name: unreachable-demo
    scrape_interval: 5s
    scrape_timeout: 2s
    metrics_path: /metrics
    static_configs: []
```

Validate and reload, then inspect active targets:

``` bash
docker compose run --rm --no-deps --entrypoint /bin/promtool prometheus check config /etc/prometheus/prometheus.yml
curl --fail --silent --show-error --request POST http://localhost:9090/-/reload
curl --fail --silent --show-error 'http://localhost:9090/api/v1/targets?state=active'
```

Expected: the failed target disappears from the active target list. Keep the empty job while querying `up{job="unreachable-demo"}` in Table view until it returns no result. Allow several former five-second scrape intervals for delayed staleness markers. Prometheus stops producing this series; it does not turn it into `1`. Its earlier zeros remain queryable at their historical times in Graph view. See [staleness](https://prometheus.io/docs/prometheus/latest/querying/basics/#staleness).

!!! note "Deleting a job can delay disappearance from queries"
    Keep the empty job until you observe the absent series. In the pinned version, deleting an entire job can cancel its delayed staleness work. If you delete it immediately or restart during cleanup, the last value can remain visible until the query lookback expires, five minutes by default. The active target list establishes that scraping has stopped even while an older sample remains queryable. See the [pinned scrape lifecycle](https://github.com/prometheus/prometheus/blob/v3.13.5/scrape/scrape.go) and [query lookback behavior](https://prometheus.io/docs/prometheus/latest/querying/basics/#staleness).

After observing the empty result, replace `prometheus/prometheus.yml` with this final configuration. This removes the empty job and the temporary timing overrides while keeping the target labels:

``` yaml
global:
  scrape_interval: 15s
  scrape_timeout: 10s
  evaluation_interval: 15s

scrape_configs:
  - job_name: prometheus
    metrics_path: /metrics
    static_configs:
      - targets: ["localhost:9090"]
        labels:
          service: prometheus
          env: local
```

``` bash
docker compose run --rm --no-deps --entrypoint /bin/promtool prometheus check config /etc/prometheus/prometheus.yml
curl --fail --silent --show-error --request POST http://localhost:9090/-/reload
curl --fail --silent --show-error --output evidence/0002-final-config.json http://localhost:9090/api/v1/status/config
curl --fail --silent --show-error --output evidence/0002-final-targets.json 'http://localhost:9090/api/v1/targets?state=active'
cat evidence/0002-final-targets.json
```

Expected: only the self-target is listed. After its next completed scrape, target metadata shows `15s` / `10s`; allow up to a fifteen-second cycle for the timing fields to update. Run both selectors in Table view:

``` promql
up{job="prometheus",service="prometheus",env="local"}
```

``` promql
up{job="unreachable-demo"}
```

The self-target should be `1`. The removed target should still return an empty result after the staleness transition you observed above. Save that current result:

``` bash
curl --fail --silent --show-error --get http://localhost:9090/api/v1/query --data-urlencode 'query=up{job="unreachable-demo"}' --output evidence/0002-removed-up.json
cat evidence/0002-removed-up.json
```

Expected: a successful response with `result: []`, rather than a sample with value `"0"`.

| Situation | Current evidence | Interpretation |
| :--- | :--- | :--- |
| Configured target, successful scrape | Target listed; `up=1` | Collection succeeded |
| Configured target, failed scrape | Target listed; `up=0`; `lastError` explains failure | Collection failed |
| Removed target after staleness | No active target; empty current selector | Collection is no longer scheduled |
| Application returns errors while metrics scrape succeeds | `up=1`; application metrics needed | Scrape health alone is insufficient |

An empty result can also mean a misspelled selector or no completed first scrape. Compare the active targets, loaded configuration, and evaluation time before concluding that a target was removed.

## 9. Active recall

Answer without reopening the configuration, then expand each answer.

??? question "A job says 5s while global.scrape_interval says 15s. Which wins?"
    The job's explicit five-second interval. After final cleanup, the job has no override and uses the global fifteen-second interval.

??? question "What controls rule evaluation frequency?"
    `global.evaluation_interval` supplies the default schedule for recording and alerting rules. A rule group can have its own interval. Neither setting controls target collection.

??? question "Why can promtool accept a target that immediately fails?"
    It validates configuration structure and constraints. A syntactically valid address can have no listener. Inspect the target's completed scrape and `lastError` to establish reachability.

??? question "How do you prove a reload took effect?"
    Read `/api/v1/status/config`, compare the intended job settings, and inspect target metadata. Runtime information gives reload status. The file contents alone do not establish the running state.

??? question "Can up=1 coexist with HTTP 500 responses from the application?"
    Yes. The metrics endpoint can scrape successfully while application requests fail. Request/status metrics provide the separate evidence needed to assess those responses.

??? question "How does removing a target differ from leaving it unreachable?"
    An unreachable target remains scheduled and produces failed scrape evidence. Removing it stops its scrapes; its series becomes stale and disappears from current instant results, while historical samples remain.

??? question "Why did the refused connection take less than the two-second timeout?"
    A timeout is an upper bound. An immediate connection error can end the attempt sooner.

## Verification and session handoff

- [ ] Explain `global`, `scrape_configs`, `static_configs`, `metrics_path`, and the resulting target labels.
- [ ] Predict the effective interval and timeout with and without a job override.
- [ ] Validate with the pinned `promtool` and prove the loaded configuration after reload.
- [ ] Interpret `up`, scrape duration, and samples scraped without treating them as business request metrics.
- [ ] Save healthy and failed target evidence and identify the exact failure from `lastError`.
- [ ] Reject an explicit timeout greater than the interval, keep the running configuration, and recover.
- [ ] Remove the failed job, distinguish empty results from zero, and restore the fifteen-second baseline.

**Files changed:** `monitoring-lab/prometheus/prometheus.yml`; evidence under `monitoring-lab/evidence/0002-*`. `docker-compose.yml` is created only if bootstrapping the shared stack. Keep notes in `monitoring-lab/evidence/0002-notes.md` explaining your predictions, observations, reload failure, and recovery.

**Versions:** Prometheus `v3.13.5`; Grafana `13.2.4`. **Final state:** one self-scrape job, `service="prometheus"`, `env="local"`, global scrape interval `15s`, timeout `10s`, and rule evaluation interval `15s`.

**Resume:** From `monitoring-lab/`, run `docker compose up -d`, check `/-/ready`, inspect active targets, and query `up{job="prometheus"}`. **Stop:** `docker compose down` preserves named volumes; keep the same Compose project when resuming.

**Authoring and learning:** The commands and observations above describe the learner exercise. Completing the authored page does not establish that you ran the lab or demonstrated understanding; use your saved evidence and recall responses to establish that.

**Next:** [Lesson 0003](0003-metric-types-and-labels.md) uses the healthy shared stack and introduces metric types and bounded label design. No instrumented application is required yet.

``` text
$teach Guide me through lesson 0003 in docs/lessons/0003-metric-types-and-labels.md.
```

| Previous Lesson | All Lessons | Next Lesson |
| :--- | :---: | ---: |
| [0001: Set Up Prometheus and Grafana Locally with Docker Compose](0001-setting-up-prometheus-and-grafana.md) | [Lesson index](index.md) | [0003: Choose Metric Types and Design Labels with Controlled Cardinality](0003-metric-types-and-labels.md) |
