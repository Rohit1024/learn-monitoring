# Prometheus & Grafana Glossary

Canonical nomenclature and standard terminology used across all lessons and guides in this observability workspace.

## Core Data Model & Primitives

**Time Series**:
A sequence of timestamped samples identified by a metric name and an identical set of labels. Samples can hold float values or native histograms.
_Avoid_: "Log stream" or "Data row".

**Sample**:
A data point with a timestamp and a float or native-histogram value. Classic histogram buckets are separate float series.
_Avoid_: "Record" or "Event".

**Label**:
A key-value pair of strings attached to a metric that defines its dimensional identity and facilitates filtering and slicing.
_Avoid_: "Tag" or "Attribute" when referring strictly to Prometheus data.

**Counter**:
A cumulative metric primitive representing a monotonically increasing counter whose value can only increase or reset to zero upon application restart.
_Avoid_: Using counters for values that can decrease (such as current memory usage).

**Gauge**:
A metric primitive representing a single numerical value that can arbitrarily go up and down over time.
_Avoid_: Using gauges when calculating rates or throughput over time intervals.

**Histogram**:
A metric instrument for observation distributions. Classic histograms expose cumulative `_bucket` series plus `_count` and `_sum`; native histograms encode the distribution within samples. Quantiles are estimates whose resolution depends on bucket layout.
_Avoid_: Averaging instance percentiles to calculate a service percentile.

**Summary**:
A metric instrument exposing observation count and sum; some client libraries also calculate configured quantiles. The Python client used in this course does not expose Summary quantiles. Count and sum can be aggregated; precomputed quantiles cannot generally be aggregated into a fleet quantile.
_Avoid_: Assuming every client library provides the same Summary features.

## Querying & Storage

**Instant Vector**:
A set of time series containing a single sample for each time series, all sharing the exact same evaluation timestamp.
_Avoid_: Confusing instant vectors with range vectors in selector syntax.

**Range Vector**:
A set of time series containing a buffer of samples over a specified past time duration for each time series (e.g., `http_requests_total[5m]`).
_Avoid_: Passing a range vector directly to a dashboard graph without an aggregation or rate function.

**TSDB (Time Series Database)**:
Prometheus's local time series storage, using an active head, a Write-Ahead Log (WAL), and persisted blocks that can be compacted into larger blocks.
_Avoid_: Treating Prometheus TSDB as a general-purpose relational or columnar SQL database.

**Write-Ahead Log (WAL)**:
An on-disk log used to recover recent series and samples after a restart. It supports recovery; it does not guarantee zero data loss under every failure.
_Avoid_: Manual editing or truncation of active WAL segment files.

**Cardinality**:
The number of distinct time series. Multiplying possible label-value counts estimates an upper bound when all combinations occur; actual cardinality depends on the combinations emitted and metric names.
_Avoid_: Adding unbounded dimensions (e.g., UUIDs, timestamps, raw URLs) to metric labels.

## Operations & Visualization

**Scrape Target**:
An endpoint exposed over HTTP(S) delivering metrics formatted according to the Prometheus exposition or OpenMetrics text format.
_Avoid_: "Polled node".

**Alertmanager**:
A dedicated service that handles alerts sent by Prometheus servers, responsible for deduplicating, grouping, silencing, and routing them to external notification receivers.
_Avoid_: Placing notification provider configs directly inside the Prometheus server YAML.

**Recording Rule**:
A server-side configured evaluation rule that precomputes frequently needed or computationally expensive PromQL expressions and saves their result as a new time series.
_Avoid_: Heavy raw dashboard queries that scan millions of samples on every browser refresh.

**Grafana Dashboard**:
A collection of visualization panels organized in a structured grid bound to one or more configured datasources using dynamic template variables.
_Avoid_: Hardcoding target hostnames or cluster endpoints directly into dashboard panel queries.

Sources: [PromQL data types](https://prometheus.io/docs/prometheus/latest/querying/basics/), [metric types](https://prometheus.io/docs/concepts/metric_types/), [Python Summary](https://prometheus.github.io/client_python/instrumenting/summary/), and [storage](https://prometheus.io/docs/prometheus/latest/storage/).

| Previous | References | Next |
| :--- | :---: | ---: |
| [Resources](resources.md) | [All references](index.md) | End of reference catalog |
