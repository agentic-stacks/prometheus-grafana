# Recording Rules

## What Are Recording Rules

Recording rules pre-compute frequently needed or computationally expensive PromQL expressions and save the result as a new time series. This provides two key benefits:

- **Dashboard performance** — Queries that aggregate over thousands of time series resolve instantly because Prometheus already computed and stored the result.
- **Simplified expressions** — Complex PromQL used in alerts or dashboards can reference a single pre-computed metric name instead of repeating the full expression.

Recording rules are evaluated at a regular interval (default: the global `evaluation_interval`) and the resulting time series is ingested back into Prometheus just like any scraped metric.

---

## Rule File Syntax

A recording rule file is a YAML file containing one or more rule groups. Each group is evaluated sequentially, and within a group rules are evaluated in order.

```yaml
groups:
  - name: <string>                    # Required — must be unique within the file
    interval: <duration>              # Optional — overrides global evaluation_interval
    limit: <int>                      # Optional — per-rule series limit (0 = no limit)
    query_offset: <duration>          # Optional — overrides global rule_query_offset
    labels:
      <labelname>: <labelvalue>       # Optional — labels added to all rules in the group
    rules:
      - record: <string>             # The new metric name (must be valid metric name)
        expr: <string>               # PromQL expression evaluated each cycle
        labels:
          <labelname>: <labelvalue>  # Optional — additional/override labels
```

The `record` field is the metric name under which the result is stored. The `expr` field is any valid PromQL expression. Optional `labels` allow you to add or override labels on the resulting time series.

---

## Naming Convention

The official Prometheus best-practices naming convention for recording rules follows the pattern:

```
level:metric:operations
```

| Component      | Meaning                                                                                      |
|----------------|----------------------------------------------------------------------------------------------|
| `level`        | The aggregation level and labels of the output (e.g., `job`, `instance_path`)                |
| `metric`       | The metric name, kept unchanged so it is easy to find in the codebase. Strip `_total` from counters only when applying `rate()` or `irate()`. |
| `operations`   | List of operations applied, newest operation first                                           |

### Rules for the operations component

- When using `rate()` or `irate()`, include the window in the operation name (e.g., `rate5m`).
- Omit `_sum` when other operations are already listed.
- Merge associative operations — `min_min` becomes `min`.
- Use `sum` as the default when no obvious single operation applies.
- For ratios of two metrics, separate them with `_per_` and name the operation `ratio`.
- When aggregating ratios, always aggregate the numerator and denominator separately, then divide. Never average ratios or averages.

### Naming examples

| Rule output                                           | Meaning                                               |
|-------------------------------------------------------|-------------------------------------------------------|
| `job:http_requests:rate5m`                            | Per-job HTTP request rate over 5m                     |
| `instance_path:requests:rate5m`                       | Per-instance-and-path request rate over 5m            |
| `path:requests:rate5m`                                | Per-path request rate (instance aggregated away)      |
| `job:request_failures_per_requests:ratio_rate5m`      | Per-job failure-to-request ratio over 5m              |
| `instance_path:request_latency_seconds:mean5m`        | Per-instance-and-path mean latency from Summary       |

---

## Examples

### Aggregate CPU usage by job

Pre-compute the average CPU usage across all instances for each job:

```yaml
groups:
  - name: cpu_aggregations
    interval: 1m
    rules:
      - record: job:node_cpu_seconds_total:avg_rate5m
        expr: |
          avg by (job) (
            rate(node_cpu_seconds_total{mode="idle"}[5m])
          )
```

### Pre-compute request rate

Compute the per-job HTTP request rate so dashboards can query the recording rule directly:

```yaml
groups:
  - name: http_aggregations
    interval: 1m
    rules:
      - record: job:prometheus_http_requests_total:rate5m
        expr: |
          sum by (job) (
            rate(prometheus_http_requests_total[5m])
          )
```

### Pre-compute error ratio

Compute the per-job error ratio (5xx responses divided by total responses), aggregating numerator and denominator separately before dividing:

```yaml
groups:
  - name: error_ratio
    interval: 1m
    rules:
      - record: job:http_requests_total:rate5m
        expr: |
          sum by (job) (
            rate(http_requests_total[5m])
          )

      - record: job:http_request_failures_total:rate5m
        expr: |
          sum by (job) (
            rate(http_request_failures_total[5m])
          )

      - record: job:http_request_failures_per_requests:ratio_rate5m
        expr: |
          job:http_request_failures_total:rate5m
            /
          job:http_requests_total:rate5m
```

---

## Load Rules

Recording rules are loaded via the `rule_files` field in `prometheus.yml`, the same mechanism used for alerting rules:

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 1m

rule_files:
  - "rules/recording_rules.yml"
  - "rules/alerting_rules.yml"
  - "rules/*.yml"               # Glob patterns are supported
```

Rule files can be reloaded at runtime without restarting Prometheus by sending `SIGHUP` to the process:

```bash
kill -HUP $(pidof prometheus)
```

Or by issuing an HTTP POST to the `/-/reload` endpoint (requires `--web.enable-lifecycle`):

```bash
curl -X POST http://localhost:9090/-/reload
```

Changes are applied only if all referenced rule files pass validation.

---

## Verify Rules

Use `promtool` to syntax-check rule files before loading them into Prometheus:

```bash
promtool check rules rules/recording_rules.yml
```

A successful check returns exit code `0` and prints the rule count:

```
Checking rules/recording_rules.yml
  SUCCESS: 5 rules found
```

Errors return exit code `1` with a description of the problem. Always run this in CI/CD pipelines before deploying rule file changes.

You can also unit-test recording rules with `promtool test rules`:

```bash
promtool test rules tests/recording_rules_test.yml
```

---

## When to Use Recording Rules

- **Dashboard performance** — Dashboards that query high-cardinality metrics across many instances benefit from pre-aggregated series. A recording rule reduces query latency from seconds to milliseconds.
- **Alert rule simplification** — When an alerting rule uses a complex or expensive expression, extract it into a recording rule. The alert expression becomes a simple threshold check against the pre-computed metric.
- **Federation** — When federating metrics to a central Prometheus, recording rules at the leaf level reduce the volume of data transferred. The central server scrapes only the pre-aggregated series.
- **Repeated expressions** — If the same PromQL expression appears in multiple dashboards or alerts, a recording rule computes it once and every consumer references the result.

---

## When NOT to Use Recording Rules

- **Simple queries** — If a query is fast and touches few time series, a recording rule adds unnecessary storage overhead with no performance benefit.
- **Rarely-used metrics** — Recording rules evaluate on every cycle regardless of whether anything queries the result. If a metric is only checked during incident investigation, compute it on demand.
- **High cardinality output** — A recording rule that preserves or increases label cardinality defeats the purpose. If `sum by (instance, path, method, status)` still produces thousands of series, the recording rule costs storage without meaningfully improving query performance.
- **Volatile expressions under development** — Avoid creating recording rules for PromQL expressions that are still being iterated on. Changes require updating the rule file and reloading Prometheus, which is slower than editing a dashboard panel.
