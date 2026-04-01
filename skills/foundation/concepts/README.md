# Metrics and Time Series Concepts

## Time Series Data Model

A time series is a sequence of measurements, ordered in time. New data is appended at regular intervals and rarely updated after initial recording. Time series enable analysis of past system states and future predictions.

Prometheus fundamentally stores all data as **streams of timestamped values belonging to the same metric and the same set of labeled dimensions**. Besides stored time series, Prometheus may generate temporary derived time series as the result of queries.

### Metric Names

A metric name specifies the general feature of a system that is measured (e.g., `http_requests_total` for the total number of HTTP requests received). Metric names:

- May use any UTF-8 characters.
- Should match the regex `[a-zA-Z_:][a-zA-Z0-9_:]*` for optimal compatibility.
- Colons are reserved for user-defined recording rules and should not be used by exporters or direct instrumentation.

### Samples

Each sample consists of:

- A **float64** or **native histogram** value.
- A **millisecond-precision timestamp**.

### Notation

The standard notation for a time series is:

```
<metric name>{<label name>="<label value>", ...}
```

For example:

```
api_http_requests_total{method="POST", handler="/messages"}
```

Alternative notations exist for metric names with UTF-8 characters outside the recommended set:

```
{"<metric name>", <label name>="<label value>", ...}
{__name__="<metric name>", <label name>="<label value>", ...}
```

## Metric Types

### Counter

A counter is a cumulative metric that represents a single **monotonically increasing** counter whose value can only increase or be reset to zero on restart.

**When to use:** Track values that only go up -- requests served, tasks completed, or errors encountered.

**When not to use:** Do not use a counter for a value that can decrease, such as the number of currently running processes. Use a gauge for that.

**Examples:**
- `http_requests_total` -- total HTTP requests received.
- `node_cpu_seconds_total` -- total seconds each CPU has spent in each mode.

### Gauge

A gauge is a metric that represents a single numerical value that can **arbitrarily go up and down**.

**When to use:** Values that fluctuate -- temperature measurements, current memory usage, or the number of concurrent requests.

**Examples:**
- `node_memory_MemAvailable_bytes` -- available memory in bytes.
- `go_goroutines` -- current number of goroutines.

### Histogram

A histogram samples observations (usually things like request durations or response sizes) and counts them in **configurable buckets**. It also provides a sum of all observed values.

A histogram with a base metric name of `<basename>` exposes the following time series during a scrape:

| Time series | Description |
|---|---|
| `<basename>_bucket{le="<upper_bound>"}` | Cumulative counter for the observation bucket. |
| `<basename>_sum` | Total sum of all observed values. |
| `<basename>_count` | Count of events observed (identical to `<basename>_bucket{le="+Inf"}`). |

Histogram buckets are **cumulative** -- each bucket contains the count of observations less than or equal to its upper bound. Use the `histogram_quantile()` function in PromQL to calculate quantiles from histograms.

Native histograms (available since Prometheus v2.40) offer dynamic buckets with improved efficiency over classic histograms.

**When to use:** Request durations, response sizes, or any observation where you need to compute quantiles or percentiles server-side.

### Summary

A summary samples observations and calculates **configurable quantiles over a sliding time window**.

A summary with a base metric name of `<basename>` exposes the following time series during a scrape:

| Time series | Description |
|---|---|
| `<basename>{quantile="<phi>"}` | Streaming phi-quantiles (0 <= phi <= 1) of observed events. |
| `<basename>_sum` | Total sum of all observed values. |
| `<basename>_count` | Count of events observed. |

**Histogram vs. Summary:** Summaries compute quantiles client-side; histograms delegate quantile computation to the server via `histogram_quantile()`. Histograms offer better aggregation capabilities across multiple instances, while summaries provide pre-calculated quantiles that cannot be aggregated.

## Jobs, Instances, and Targets

Prometheus scrapes metrics from monitored **targets**. Each target exposes metrics at an HTTP endpoint.

- **Instance:** An endpoint you can scrape, usually corresponding to a single process.
- **Job:** A collection of instances with the same purpose -- a process replicated for scalability or reliability.

Example job structure:

```
job: api-server
  instance 1: 1.2.3.4:5670
  instance 2: 1.2.3.4:5671
  instance 3: 5.6.7.8:5670
  instance 4: 5.6.7.8:5671
```

### Automatically Generated Labels and Time Series

When Prometheus scrapes a target, it automatically attaches two labels:

- **`job`:** The configured job name that the target belongs to.
- **`instance`:** The `<host>:<port>` part of the target's URL that was scraped.

Prometheus also stores the following time series for each instance scrape:

| Metric | Description |
|---|---|
| `up{job="<job>", instance="<instance>"}` | `1` if the instance is reachable, `0` if the scrape failed. |
| `scrape_duration_seconds` | Duration of the scrape. |
| `scrape_samples_post_metric_relabeling` | Number of samples remaining after metric relabeling. |
| `scrape_samples_scraped` | Number of samples the target exposed. |
| `scrape_series_added` | Approximate number of new series in this scrape (since v2.10). |

## Labels

Labels are key-value pairs that enable Prometheus's dimensional data model. Any given combination of labels for the same metric name identifies a particular dimensional instantiation of that metric. Any change in label values, including adding or removing labels, creates a new time series.

### Label Naming

- Label names may use any UTF-8 characters.
- Label names should match the regex `[a-zA-Z_][a-zA-Z0-9_]*` for best compatibility.
- Label names beginning with `__` (two underscores) are **reserved for internal use** by Prometheus.
- Empty label values are treated as equivalent to non-existent labels.

### Reserved Labels

| Label | Purpose |
|---|---|
| `__name__` | Holds the metric name. The notation `metric_name{...}` is syntactic sugar for `{__name__="metric_name", ...}`. |
| `job` | Automatically attached by Prometheus; identifies the configured job name of the scraped target. |
| `instance` | Automatically attached by Prometheus; identifies the `<host>:<port>` of the scraped target. |

### Label Cardinality

Every unique combination of metric name and label key-value pairs constitutes a distinct time series. High cardinality labels (labels with many possible values, such as user IDs, email addresses, or unbounded request paths) create large numbers of time series. This directly increases Prometheus memory and storage usage and can degrade query performance. Keep label values bounded and of low cardinality.

## PromQL Preview

Prometheus provides a functional query language called **PromQL** (Prometheus Query Language) that lets users select and aggregate time series data in real time. PromQL supports filtering by labels, aggregating across dimensions, and applying functions such as `rate()`, `histogram_quantile()`, and arithmetic operations.

Detailed PromQL coverage -- including selectors, operators, functions, and common query patterns -- is provided in the **operations/promql** skill.
