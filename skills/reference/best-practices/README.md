# Best Practices

Conventions and guidelines from the official Prometheus documentation for naming metrics, instrumenting applications, configuring alerts, writing recording rules, managing labels, and building dashboards.

Sources:
- https://prometheus.io/docs/practices/naming/
- https://prometheus.io/docs/practices/instrumentation/
- https://prometheus.io/docs/practices/histograms/
- https://prometheus.io/docs/practices/alerting/
- https://prometheus.io/docs/practices/rules/

---

## 1. Metric Naming

Metric names follow the pattern **`<namespace>_<name>_<unit>`** where:

- **Namespace**: A single-word application or domain prefix (e.g., `prometheus`, `http`, `process`).
- **Name**: A descriptive identifier of what is being measured.
- **Unit**: The base unit as a plural suffix.

### Rules

- Names must comply with the Prometheus data model for valid characters (`[a-zA-Z_:][a-zA-Z0-9_:]*`).
- A metric must refer to a single unit and a single quantity. Never mix seconds with milliseconds in the same metric.
- Always use base units. Let the display layer handle human-friendly formatting.
- A metric should represent the same logical measurement across all of its label dimensions.

### Special Suffixes

| Suffix | Usage | Example |
|--------|-------|---------|
| `_total` | Accumulating counter | `http_requests_total` |
| `_info` | Metadata pseudo-metric (value always 1) | `foobar_build_info` |
| `_bytes` | Data sizes | `http_response_size_bytes` |
| `_seconds` | Durations | `http_request_duration_seconds` |
| `_seconds_total` | Accumulating time counter | `process_cpu_seconds_total` |
| `_timestamp_seconds` | Tracked pipeline timestamps | `pipeline_last_run_timestamp_seconds` |

### Base Units

| Dimension | Base Unit |
|-----------|-----------|
| Time | seconds |
| Data | bytes |
| Temperature | celsius |
| Length | meters |
| Percent / ratio | ratio (0-1, not 0-100) |
| Voltage | volts |
| Current | amperes |
| Energy | joules |
| Mass | grams |

### Examples

```text
# Good
prometheus_notifications_total
http_request_duration_seconds
node_memory_usage_bytes
process_cpu_seconds_total

# Bad
http_request_duration_milliseconds   # use seconds, not milliseconds
request_count                        # missing namespace, missing _total
```

---

## 2. Instrumentation Guidelines

The core principle is **instrument everything**. Every library, subsystem, and service should expose metrics that reveal its performance characteristics.

### Online-Serving Systems

Services that handle synchronous requests (HTTP APIs, databases, RPC servers). Key metrics:

| Metric | Description |
|--------|-------------|
| Request count | Number of completed requests (`_total` counter) |
| Error rate | Number of failed requests (`_total` counter) |
| Latency | Response time distribution (histogram) |
| In-progress requests | Currently active requests (gauge) |

Count queries at completion (not initiation) so that error and latency statistics align. Monitor both the client side and the server side.

### Offline Processing

Queue-based or pipeline systems where the caller does not wait for a response. Per-stage metrics:

| Metric | Description |
|--------|-------------|
| Items in | Counter of items entering the stage |
| Items out | Counter of items leaving the stage |
| In-progress | Gauge of items currently being processed |
| Last processing timestamp | Gauge of the most recent processing time |
| Errors | Counter of failures per stage |

Use heartbeat mechanisms (dummy items with insertion timestamps) to detect stalled stages and measure end-to-end propagation duration.

### Batch Jobs

Non-continuous jobs that are difficult to scrape during execution. Push results to a PushGateway:

| Metric | Description |
|--------|-------------|
| Last success timestamp | Gauge pushed at the end of a successful run |
| Job duration | Gauge of total runtime in seconds |
| Records processed | Counter of items handled |
| Last completion time | Gauge, regardless of success or failure |

For jobs running longer than a few minutes, add pull-based scraping to capture resource usage and internal latency during execution.

### The Four Golden Signals

Derived from Google SRE practices, these four signals cover the essential health indicators for any service:

| Signal | What to Measure | Metric Type |
|--------|----------------|-------------|
| **Latency** | Time to service a request (separate successful vs. failed) | Histogram |
| **Traffic** | Demand on the system (requests/sec, sessions, transactions) | Counter |
| **Errors** | Rate of failed requests (explicit, implicit, policy-based) | Counter |
| **Saturation** | How full the system is (CPU, memory, I/O, queue depth) | Gauge |

### Subsystem Checklist

- **Libraries**: Expose query counts, errors, and latency for every external resource call.
- **Logging**: Add a counter for every log line; correlate log messages with metric increments.
- **Failures**: Always pair a failure counter with an attempt counter to calculate failure ratios.
- **Thread pools**: Expose queued requests, active threads, total threads, tasks processed, and queue wait time.
- **Caches**: Track total queries, hit count, overall latency, and underlying resource metrics.
- **Custom collectors**: Include a duration gauge and an error counter.

### General Guidelines

- Keep label cardinality below 10 for most metrics.
- Export timestamps rather than elapsed time so that `time() - my_metric` calculations remain accurate.
- Avoid procedurally generated metric names; use labels instead.
- Export zero-valued defaults for metrics known to exist.
- Instantiate metrics in the same file where they are used for easier debugging.

---

## 3. Histogram vs Summary

Histograms and summaries both track distributions, but they differ in critical ways.

### Comparison

| Aspect | Histogram | Summary |
|--------|-----------|---------|
| **Aggregatability** | Fully aggregatable across instances using `histogram_quantile()` | Not aggregatable; averaging quantiles is statistically meaningless |
| **Quantile accuracy** | Controlled by bucket width (observed-value dimension); bucket boundaries determine error | Controlled by configurable phi-tolerance (0 <= phi <= 1) |
| **Client cost** | Minimal; increments bucket counters only | Expensive; requires streaming quantile calculation |
| **Server cost** | Higher; quantiles computed at query time | Minimal; quantiles pre-computed on the client |
| **Configuration** | Choose bucket boundaries suitable for expected value range | Choose desired phi-quantiles and sliding window; immutable after deployment |
| **Time series** | One series per bucket plus `_sum` and `_count` | One series per quantile plus `_sum` and `_count` |

### When to Choose Each

**Choose histograms when:**
- You need to aggregate quantiles across multiple instances (the most common case in distributed systems).
- You have a reasonable understanding of the expected value range to pick appropriate buckets.
- You want to calculate arbitrary quantiles at query time without redeploying.

**Choose summaries when:**
- You need accurate quantiles for a single instance regardless of value distribution.
- You do not need to aggregate across instances.
- You want to minimize server-side query cost.

### Default Histogram Buckets

The Prometheus client libraries provide default buckets suitable for typical HTTP request durations:

```text
.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10
```

Customize buckets for your specific use case. For example, if most requests complete in 100-500ms, add more granularity in that range.

---

## 4. Alerting Best Practices

Core philosophy: **Keep alerting simple, alert on symptoms, have good consoles to allow pinpointing causes, and avoid having pages where there is nothing to do.**

### Alert on Symptoms, Not Causes

Focus on conditions that directly impact users rather than trying to anticipate every possible failure mode. A single symptom-based alert (e.g., high error rate at the load balancer) catches many different underlying causes.

### Severity Levels

| Level | Action | Criteria |
|-------|--------|----------|
| **Critical / Page** | Wake someone up; requires immediate human intervention | User-visible impact is occurring or imminent |
| **Warning / Ticket** | Create a ticket for next-business-day attention | Degradation that will become critical if unaddressed |
| **Info** | Log or dashboard only; no notification | Useful context for investigations but not actionable on its own |

### Domain-Specific Guidance

| Domain | Guidance |
|--------|----------|
| **Online services** | Page on user-facing error rates and latency at the highest stack level. If lower-level failures do not affect end users, do not page separately. |
| **Offline processing** | Alert when throughput delays approach the threshold where users would be impacted. |
| **Batch jobs** | Page only if a job misses its success window by 2+ full run cycles (e.g., 10 hours for a 4-hour job). Never page on a single failure. |
| **Capacity** | Monitor approaching limits to enable proactive intervention before saturation. |
| **Metamonitoring** | Verify the health of the monitoring infrastructure itself (Prometheus, Alertmanager, PushGateway). Use integration tests to confirm the alert delivery chain. |

### Reducing Alert Noise

- Allow slack in thresholds so minor fluctuations do not trigger pages.
- Prefer end-to-end blackbox tests that measure the full alert flow over individual component alerts.
- Every page should be actionable. If there is nothing an on-call engineer can do, the alert should not page.
- Link alerts to relevant dashboards and runbooks so the responder can quickly identify the root cause.

### Alert Naming

The community convention is to use CamelCase for alert names (e.g., `HighRequestLatency`, `TargetDown`).

---

## 5. Recording Rule Conventions

Recording rules pre-compute frequently used or expensive PromQL expressions and store the results as new time series.

### Naming Convention: `level:metric:operations`

| Component | Description | Example |
|-----------|-------------|---------|
| **level** | The aggregation level and labels of the output | `instance_path`, `job`, `path` |
| **metric** | The original metric name; strip `_total` when using `rate()` or `irate()` | `requests`, `request_latency_seconds` |
| **operations** | Applied operations, newest first | `rate5m`, `ratio_rate5m`, `mean5m` |

### Rules for the Naming Components

- Omit `_sum` from the metric component if other operations are present.
- Merge associative operations (`min_min` becomes `min`).
- Use `sum` as the default when no obvious operation applies.
- Use `_per_` and `ratio` for division operations.
- When dividing `_count` and `_sum` of a summary, replace `rate` with `mean` in the operation name.

### Aggregation Guidance

- **Ratios**: Aggregate numerator and denominator separately, then divide. Never average a ratio.
- **Averages**: Never average an average; it is statistically invalid.
- **Label preservation**: Use `without` clauses specifying the labels being removed, rather than `by`, to preserve useful labels like `job`.

### Examples

Per-second rate with path labels:

```yaml
- record: instance_path:requests:rate5m
  expr: rate(requests_total{job="myjob"}[5m])

- record: path:requests:rate5m
  expr: sum without (instance)(instance_path:requests:rate5m{job="myjob"})
```

Request failure ratio:

```yaml
- record: instance_path:request_failures_per_requests:ratio_rate5m
  expr: |2
    instance_path:request_failures:rate5m{job="myjob"}
  /
    instance_path:requests:rate5m{job="myjob"}
```

Average latency from a summary:

```yaml
- record: instance_path:request_latency_seconds:mean5m
  expr: |2
    instance_path:request_latency_seconds_sum:rate5m{job="myjob"}
  /
    instance_path:request_latency_seconds_count:rate5m{job="myjob"}
```

### When to Create Recording Rules

- When a PromQL expression is used in multiple dashboards or alerts.
- When a query is too expensive to run at every evaluation (e.g., aggregating across many series).
- When you need to feed one rule into another (chained aggregation).

---

## 6. Label Best Practices

Labels add dimensions to metrics but each unique label combination creates a new time series. Cardinality management is critical.

### Keep Cardinality Low

- Aim for fewer than 10 distinct values per label in most metrics.
- Never use unbounded, high-cardinality values as labels: user IDs, email addresses, IP addresses, full URLs, or UUIDs.
- Each additional label value multiplies the total number of time series.

### Label Design Rules

- Use labels to differentiate characteristics of the thing being measured:
  ```text
  api_http_requests_total{method="GET", endpoint="/users"}
  ```
- Never embed label names into the metric name. Use `api_http_requests_total` with a `method` label, not `api_http_requests_get_total`.
- When a label is aggregated away with `sum()`, the resulting metric should still be meaningful.

### metric_relabel_configs

Use `metric_relabel_configs` in the scrape configuration to control label cardinality at ingestion time:

```yaml
scrape_configs:
  - job_name: myapp
    metric_relabel_configs:
      # Drop a high-cardinality label before ingestion
      - source_labels: [__name__]
        regex: "expensive_metric_.*"
        action: drop

      # Replace high-cardinality label values with a fixed value
      - source_labels: [user_id]
        target_label: user_id
        regex: ".*"
        replacement: "redacted"
```

### Cardinality Estimation

Estimate total time series as the product of all label value counts:

```text
series = metric_count * label_A_values * label_B_values * ...
```

A metric with 3 labels of 10 values each produces 1,000 series. Add a fourth label with 100 values and it becomes 100,000.

---

## 7. Dashboard Best Practices

Effective dashboards surface actionable insights. Two standard methodologies provide structure.

### USE Method (for Resources)

Applicable to infrastructure components (CPU, memory, disk, network):

| Signal | What to Measure | Example Query |
|--------|----------------|---------------|
| **Utilization** | Proportion of resource consumed | `rate(node_cpu_seconds_total{mode!="idle"}[5m])` |
| **Saturation** | Work the resource cannot service (queue depth) | `node_load1 - count(node_cpu_seconds_total{mode="idle"})` |
| **Errors** | Count of error events | `rate(node_disk_io_errs_total[5m])` |

### RED Method (for Services)

Applicable to request-driven services (APIs, microservices):

| Signal | What to Measure | Example Query |
|--------|----------------|---------------|
| **Rate** | Requests per second | `rate(http_requests_total[5m])` |
| **Errors** | Failed requests per second | `rate(http_requests_total{status=~"5.."}[5m])` |
| **Duration** | Distribution of response times | `histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))` |

### Dashboard Organization

- **Top level**: Service overview with RED metrics. This is the first dashboard an on-call engineer opens.
- **Second level**: Subsystem dashboards (database, cache, queue) with USE and RED metrics for each component.
- **Third level**: Detailed debugging dashboards with per-instance metrics, logs, and traces.

### Variables and Templating

Use Grafana template variables to make dashboards reusable:

```text
$job       - Select the Prometheus job
$instance  - Filter to a specific instance
$interval  - Adjust the rate() range dynamically
```

Define variables from label values using queries like `label_values(up, job)` so dashboards adapt automatically as infrastructure changes.

### Linking

- Link alert rules to the relevant dashboard so that alert notifications include a direct URL.
- Add drill-down links from overview panels to detailed dashboards.
- Include links to runbooks in dashboard annotations and alert descriptions.
- Use Grafana's data links feature to connect panels to log explorers or trace UIs with pre-filled query parameters.

### General Tips

- Set appropriate time ranges: 1-hour for real-time operations, 24-hour for daily patterns, 7-day for capacity planning.
- Use consistent color schemes across dashboards (e.g., red for errors, green for success).
- Add threshold lines on panels to show SLO targets visually.
- Prefer `rate()` over raw counters on dashboard panels to normalize across scrape intervals.
- Place the most important panels at the top of the dashboard since that is where attention goes first during incidents.
