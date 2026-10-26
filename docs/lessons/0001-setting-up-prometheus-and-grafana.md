---
icon: lucide/server
---

# 0001: Set Up Prometheus and Grafana Locally with Docker Compose

**Time:** 60–90 minutes. **Prerequisites:** a terminal, `curl`, and a running container engine with Docker Compose support. This is a local learning stack. You will create the files below during the exercise.

By the end, you should be able to explain the metrics pipeline, start Prometheus and Grafana with persistent storage, verify both HTTP services, and find Prometheus self-monitoring data.

## Understand the pipeline

``` mermaid
flowchart TD
    App["Application updates counters, gauges, and histograms"]
    Endpoint["Application exposes /metrics"]
    Scraper["Prometheus pulls /metrics on a schedule"]
    Store[("Prometheus time series storage")]
    Query["PromQL engine and HTTP query API"]
    Browser["Prometheus expression browser"]
    Grafana["Grafana Explore and dashboard panels"]
    App --> Endpoint
    Endpoint -->|"Scraped samples"| Scraper
    Scraper --> Store
    Store -->|"Selected and computed results"| Query
    Query --> Browser
    Query --> Grafana
```

The arrows show data flow. Prometheus initiates the scrape request; Grafana initiates query requests. An application's `/metrics` endpoint exposes current instrument values; Prometheus stores the history. Grafana queries that history to draw panels. We add the application in lesson 0004; today Prometheus scrapes itself.

A series is identified by its metric name and labels. For example, `up{job="prometheus",instance="localhost:9090"}` describes scrape health for one target. Its value is 1 for a successful scrape and 0 for a failed scrape. This does not tell us whether the application's business responses are correct. See [jobs and instances](https://prometheus.io/docs/concepts/jobs_instances/).

## 1. Check prerequisites and create the workspace

``` bash
docker version
docker compose version
curl --version
mkdir -p monitoring-lab/prometheus
cd monitoring-lab
```

`docker version` should show a reachable server as well as a client. If your `docker` command is a Podman compatibility wrapper, confirm that its VM and Compose provider are running before continuing. A refused engine socket is an environment problem, not a Prometheus configuration error.

All remaining shell commands run from `monitoring-lab/`:

``` text
monitoring-lab/
├── docker-compose.yml
└── prometheus/
    └── prometheus.yml
```

!!! note "Pinned versions"
    This lesson uses Prometheus `v3.13.5` (listed as LTS) and Grafana `13.2.4`, checked against the official [Prometheus downloads](https://prometheus.io/download/) and [Grafana downloads](https://grafana.com/grafana/download) on 2026-10-23. These are reproducible course choices; recheck releases when updating the course. UI menu names can differ across versions.

## 2. Configure Prometheus

Save this as `prometheus/prometheus.yml`:

``` yaml
global:
  scrape_interval: 15s  # (1)!
  scrape_timeout: 10s
  evaluation_interval: 15s  # (2)!

scrape_configs:
  - job_name: prometheus
    metrics_path: /metrics
    static_configs:
      - targets: ["localhost:9090"]  # (3)!
```

1. Prometheus attempts a scrape every 15 seconds. The timeout must not exceed the interval.
2. Recording and alerting rules use this evaluation schedule. There are no rules in this lesson.
3. This address is evaluated inside the Prometheus container, so `localhost` correctly refers to Prometheus itself.

The interval is a lab choice, not a universal production recommendation. It trades measurement resolution against scrape/storage overhead. Read the [configuration reference](https://prometheus.io/docs/prometheus/latest/configuration/configuration/) for exact field semantics.

## 3. Define the services

Save this as `docker-compose.yml`:

``` yaml
services:
  prometheus:
    image: prom/prometheus:v3.13.5
    restart: unless-stopped
    ports:
      - "127.0.0.1:9090:9090"
    volumes:
      - ./prometheus:/etc/prometheus:ro  # (1)!
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
      - prometheus  # (2)!

networks:
  monitoring:
    driver: bridge

volumes:
  prometheus_data: {}
  grafana_data: {}
```

1. The read-only directory mount supplies configuration. Named volumes separately hold Prometheus history and Grafana's database.
2. This specifies startup order. It does not prove Prometheus is ready to serve queries; check readiness explicitly below.

Ports bind to the host's loopback interface for this local lab. The initial Grafana login is `admin` / `admin`; follow its first-login password prompt. Bootstrap credentials only apply when initializing a fresh Grafana database.

`--web.enable-lifecycle` enables the HTTP reload and quit endpoints. Later lessons use configuration reload; it is not needed for normal scraping. The [management API](https://prometheus.io/docs/prometheus/latest/management_api/) documents these endpoints.

## 4. Validate, start, and check readiness

Before startup, check Compose syntax and the Prometheus configuration:

``` bash
docker compose config --quiet
docker compose run --rm --no-deps --entrypoint /bin/promtool prometheus check config /etc/prometheus/prometheus.yml
```

Expected: Compose exits successfully, and `promtool` reports a valid configuration. The first run downloads the pinned Prometheus image.

``` bash
docker compose up -d
docker compose ps
curl --fail --silent --show-error http://localhost:9090/-/ready
curl --fail --silent --show-error http://localhost:3000/api/health
```

Expected: both containers are running, Prometheus reports readiness, and Grafana's JSON health response includes `"database": "ok"`. Repeat the HTTP checks after a short startup delay if necessary. These are expected lab observations, not evidence that this stack has already been started on your computer.

A running process and a ready HTTP service are different observations. If either check continues to fail, inspect the relevant logs:

``` bash
docker compose logs --tail=60 prometheus
docker compose logs --tail=60 grafana
```

## 5. Find your first metrics in the Prometheus UI

Open [Prometheus](http://localhost:9090). Find the target listing (often under Status or Monitoring, depending on version). Confirm the `prometheus` job has endpoint `http://localhost:9090/metrics` and reaches `UP` after a scrape.

In the query/expression page, run these separately in **Table** view:

``` promql
up{job="prometheus"}
```

``` promql
scrape_duration_seconds{job="prometheus"}
```

The first should return 1 after a successful scrape. The second measures scrape duration in seconds. Switch the `up` expression to **Graph**, select the last 15 minutes, and observe samples accumulating. Table evaluates at one time; Graph repeats evaluation across a time range. [PromQL basics](https://prometheus.io/docs/prometheus/latest/querying/basics/) explains this distinction.

Reproduce the health query through the HTTP API:

``` bash
curl --fail --silent --show-error --get http://localhost:9090/api/v1/query --data-urlencode 'query=up{job="prometheus"}'
```

Expected JSON includes `"status":"success"`, `"resultType":"vector"`, the target's labels, and a value string `"1"`. This proves query access as well as scraping; later sessions explain the response structure.

## 6. Inspect Grafana and understand the addresses

Open [Grafana](http://localhost:3000), log in, and locate Dashboards, Explore, and Connections/Data sources. Lesson 0011 covers creating the Prometheus connection and reproducing queries in Explore; today's endpoint is a healthy Grafana service.

Predict the address each caller needs before reading the explanation:

- Your host browser reaches Prometheus at `http://localhost:9090` through the published port.
- Grafana's server reaches Prometheus at `http://prometheus:9090` using Compose service discovery.
- Prometheus reaches its own endpoint at `http://localhost:9090/metrics` inside its container.

Inside Grafana, `localhost:9090` would point back to Grafana's container. Review [Compose networking](https://docs.docker.com/compose/how-tos/networking/) if these addresses are unclear.

## 7. Failure exercise and independent challenge

**Guided failure:** If startup reports that host port 9090 is already in use, inspect your running services. Change the mapping to `127.0.0.1:9091:9090` and run `docker compose up -d` again. Use `http://localhost:9091` from the host; the container port and future Grafana URL stay at 9090. Restore the original mapping when the conflict is resolved.

**Independent challenge:** Save a successful API query response and its time. Stop and recreate the containers with the commands below, then find the earlier time in the Prometheus graph. Explain which files and volumes make this possible. Do not equate a short scrape gap during downtime with lost historical data.

``` bash
docker compose down
docker compose up -d
```

`down` without `--volumes` preserves these named volumes. Use the same project directory so Compose uses the same project and volume names.

## 8. Active recall

??? question "Where does the metric history live?"
    In Prometheus's time series storage, backed here by `prometheus_data`. The application's endpoint exposes instrument values; Grafana queries the stored history.

??? question "Does up=1 prove an application returns successful responses?"
    No. It proves the scrape succeeded. Application errors need separate request/status metrics, which later sessions add.

??? question "Why does Grafana use http://prometheus:9090?"
    Its server sends the query from inside the Compose network. Service discovery resolves `prometheus` to that service; `localhost` would refer to Grafana's own container.

??? question "What does evaluation_interval change?"
    The schedule for evaluating recording and alerting rules. `scrape_interval` controls metric collection.

??? question "Does docker compose ps prove readiness?"
    It reports container state. This configuration has no Compose health checks, so readiness requires the explicit HTTP checks.

## Verification and session handoff

- [ ] Explain application exposition, Prometheus collection/storage/querying, and Grafana visualization without looking at the diagram.
- [ ] Validate Compose and Prometheus configuration.
- [ ] Verify both HTTP health endpoints and `up{job="prometheus"}=1`.
- [ ] Reproduce a query through the HTTP API.
- [ ] Recover from a port conflict and explain host versus container ports.
- [ ] Recreate containers while preserving history.

**Files created:** `monitoring-lab/docker-compose.yml` and `monitoring-lab/prometheus/prometheus.yml`.

**Resume:** From `monitoring-lab/`, run `docker compose up -d`, repeat the HTTP checks, and open the UIs. **Stop:** `docker compose down` preserves named volumes. Record actual versions and your observations before moving on.

**Next session request:**

``` text
$teach Expand lesson 0002 using the outline in docs/lessons/0002-prometheus-configuration-and-scraping.md and docs/lesson-authoring.md.
```

| Previous Lesson | All Lessons | Next Lesson |
| :--- | :---: | ---: |
| First lesson | [Session catalog](index.md) | [0002: Configure Prometheus Scrape Jobs, Intervals, and Target Health](0002-prometheus-configuration-and-scraping.md) |
