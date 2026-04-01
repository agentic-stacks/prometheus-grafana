# Diagnose: Prometheus

Troubleshooting guide for Prometheus. Each section follows a decision-tree format:
**If symptom -> check cause -> if confirmed -> apply fix.**

**Also check:** `skills/reference/known-issues/` for version-specific problems.

All endpoints, commands, and queries are exact and drawn from official Prometheus documentation.

---

## 1. Health Check

Prometheus exposes two lifecycle endpoints (no flags required):

| Endpoint | Methods | Meaning |
|---|---|---|
| `GET /-/healthy` | GET, HEAD | Returns **200** as long as the Prometheus process is running. Does **not** indicate readiness to serve queries. |
| `GET /-/ready` | GET, HEAD | Returns **200** only when Prometheus has finished startup (WAL replay, initial scrape cycle) and is ready to serve traffic. Returns **503** during startup. |

### Decision Tree

```
If Prometheus container/process appears running
  -> curl -sf http://localhost:9090/-/healthy
     -> If non-200 or connection refused
        -> Process is not running. Check logs: docker logs <container> or journalctl -u prometheus
     -> If 200
        -> curl -sf http://localhost:9090/-/ready
           -> If 503
              -> Prometheus is still starting (WAL replay or loading blocks).
                 Check startup progress in logs. See Section 5 (WAL Issues).
           -> If 200
              -> Prometheus is healthy and ready to serve queries.
```

### Reload and Shutdown

These require the `--web.enable-lifecycle` flag:

| Endpoint | Methods | Purpose |
|---|---|---|
| `PUT /-/reload` or `POST /-/reload` | PUT, POST | Triggers configuration and rule file reload. Alternative: `kill -SIGHUP <prometheus_pid>` |
| `PUT /-/quit` or `POST /-/quit` | PUT, POST | Triggers graceful shutdown. Alternative: `kill -SIGTERM <prometheus_pid>` |

Reload command:

```bash
curl -X POST http://localhost:9090/-/reload
```

If lifecycle endpoints are not enabled, use signals:

```bash
kill -SIGHUP $(pidof prometheus)
```

---

## 2. Targets Not Being Scraped

### Symptom: Target shows state DOWN

```
If target shows health=DOWN in the Targets page (http://localhost:9090/targets)
  -> Check the "Last Error" column for the exact error message.
     -> If error contains "connection refused"
        -> The target process is not listening on the expected port.
           Fix: Verify the target application is running and bound to the correct interface.
           Test: curl -sf http://<target_host>:<target_port>/metrics
     -> If error contains "context deadline exceeded"
        -> Scrape is timing out.
           Fix: Increase scrape_timeout in scrape_config (default is 10s).
           Test: time curl -s http://<target_host>:<target_port>/metrics > /dev/null
     -> If error contains "no route to host" or "i/o timeout"
        -> Network or firewall issue.
           Fix: Check firewall rules, security groups, network policies.
           Test: nc -zv <target_host> <target_port>
     -> If error contains "server returned HTTP status 401" or "403"
        -> Authentication required.
           Fix: Add authorization config to the scrape_config:
           authorization:
             credentials_file: /path/to/token
     -> If error contains "bad certificate" or "x509"
        -> TLS verification failure.
           Fix: Configure tls_config in scrape_config:
           tls_config:
             ca_file: /path/to/ca.crt
             insecure_skip_verify: false  # only set true for debugging
```

### Symptom: Target not appearing at all

```
If expected target is not listed on the Targets page
  -> Check if the scrape_config exists in the running config:
     curl -s http://localhost:9090/api/v1/status/config | python3 -m json.tool
     -> If the job is missing from the config
        -> The configuration file does not include this job.
           Fix: Add the scrape_config and reload.
     -> If the job exists in the config
        -> Check the Service Discovery page: http://localhost:9090/service-discovery
           -> If the target appears in discovered targets but not in active targets
              -> A relabel_configs rule is dropping it.
                 Fix: Review relabel_configs for action: drop rules.
                 Debug: Temporarily remove relabel_configs and reload to confirm.
           -> If the target does not appear in discovered targets
              -> Service discovery is not finding it.
                 Fix (static_configs): Verify the targets list has the correct host:port.
                 Fix (dns_sd_configs): Verify DNS resolution: dig SRV <record>
                 Fix (kubernetes_sd_configs): Check RBAC permissions for the Prometheus service account.
                 Fix (file_sd_configs): Verify the JSON/YAML file exists and is valid.
  -> After any config change, reload:
     curl -X POST http://localhost:9090/-/reload
     # or
     kill -SIGHUP $(pidof prometheus)
```

### Verify via API

List all active targets and their states:

```bash
curl -s http://localhost:9090/api/v1/targets?state=active | python3 -m json.tool
```

List only dropped targets:

```bash
curl -s http://localhost:9090/api/v1/targets?state=dropped | python3 -m json.tool
```

Filter by scrape pool:

```bash
curl -s "http://localhost:9090/api/v1/targets?scrapePool=my-job" | python3 -m json.tool
```

---

## 3. High Cardinality

### Symptom: Excessive memory usage or slow queries

```
If Prometheus uses more memory than expected or queries are slow
  -> Check TSDB status for cardinality breakdown:
     curl -s http://localhost:9090/api/v1/status/tsdb | python3 -m json.tool
     -> Response includes:
        - seriesCountByMetricName: top metrics by series count
        - labelValueCountByLabelName: labels with most distinct values
        - memoryInBytesByLabelName: memory consumed per label name
        - seriesCountByLabelValuePair: label-value pairs with most series
     -> If a single metric has hundreds of thousands of series
        -> That metric has too many label combinations.
           Identify it:
           curl -s "http://localhost:9090/api/v1/status/tsdb?limit=20" | python3 -m json.tool
        -> Fix: Use metric_relabel_configs to drop unnecessary label values or the entire metric:

           scrape_configs:
             - job_name: 'my-job'
               metric_relabel_configs:
                 # Drop a specific high-cardinality metric entirely
                 - source_labels: [__name__]
                   regex: 'expensive_metric_name'
                   action: drop
                 # Drop a high-cardinality label from all metrics
                 - regex: 'high_cardinality_label_name'
                   action: labeldrop
                 # Keep only specific label values
                 - source_labels: [method]
                   regex: '(GET|POST|PUT|DELETE)'
                   action: keep

     -> If a label like "instance_id" or "pod" has millions of values
        -> This label is generating combinatorial explosion.
           Fix: Drop or aggregate the label using metric_relabel_configs (see above).
```

### Find high-cardinality metrics with PromQL

Top 10 metrics by series count:

```promql
topk(10, count by (__name__)({__name__=~".+"}))
```

Total number of active time series:

```promql
prometheus_tsdb_head_series
```

Rate of series creation (churn indicator):

```promql
rate(prometheus_tsdb_head_series_created_total[5m])
```

### Cardinality guidelines (from official best practices)

- Keep label value cardinality below 10 per label as a general guideline.
- If a metric has cardinality over 100 or the potential to grow that large, investigate alternate solutions.
- Never procedurally generate metric names; use labels instead.
- High cardinality creates overhead across RAM, CPU, disk I/O, and network.

---

## 4. Out of Memory (OOM)

### Symptom: Prometheus process killed by OOM killer

```
If Prometheus is killed by the OOM killer
  -> Verify OOM kill in system logs:
     dmesg | grep -i "oom.*prometheus"
     # or
     journalctl -k | grep -i "oom.*prometheus"

  -> Check total active series count:
     curl -s http://localhost:9090/api/v1/query?query=prometheus_tsdb_head_series | python3 -m json.tool
     -> If series count is in the millions
        -> Cardinality is the primary cause. Follow Section 3 (High Cardinality).

  -> Check memory usage by component:
     curl -s http://localhost:9090/api/v1/query?query=process_resident_memory_bytes | python3 -m json.tool

  -> Check head chunks memory:
     curl -s "http://localhost:9090/api/v1/query?query=prometheus_tsdb_head_chunks_storage_size_bytes" | python3 -m json.tool

  -> If series count is reasonable but memory is still high
     -> Check retention settings:
        # Current retention is set via --storage.tsdb.retention.time (default 15d)
        # or --storage.tsdb.retention.size
        Fix: Reduce retention:
          --storage.tsdb.retention.time=7d
        Fix: Set a size-based retention limit:
          --storage.tsdb.retention.size=10GB

  -> If the problem recurs after reducing cardinality and retention
     -> Increase memory limits for the container/process.
     -> Consider federation or remote_write to offload storage.
     -> Shard scrape targets across multiple Prometheus instances.
```

### Monitor to prevent OOM

Set up alerting on these metrics:

```yaml
groups:
  - name: prometheus-self-monitoring
    rules:
      - alert: PrometheusHighMemoryUsage
        expr: process_resident_memory_bytes{job="prometheus"} / on() machine_memory_bytes > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Prometheus memory usage above 80%"

      - alert: PrometheusHighCardinality
        expr: prometheus_tsdb_head_series > 2000000
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Prometheus head series count exceeds 2 million"

      - alert: PrometheusSeriesChurn
        expr: rate(prometheus_tsdb_head_series_created_total[1h]) > 1000
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "High series churn rate detected"
```

---

## 5. WAL Issues

### Symptom: Slow startup

```
If Prometheus takes a long time to start (/-/ready returns 503 for extended periods)
  -> Check logs for WAL replay messages:
     docker logs <container> 2>&1 | grep -i "wal"
     # Look for: "Replaying WAL, this may take a while"

  -> Check WAL directory size:
     du -sh /prometheus/wal
     # or inside container:
     docker exec <container> du -sh /prometheus/wal
     -> If WAL is many gigabytes
        -> WAL has grown due to high ingestion rate or incomplete compaction.

        -> Fix: Enable WAL compression (reduces WAL size by ~50%):
           --storage.tsdb.wal-compression
           # This is enabled by default since Prometheus 2.20+

        -> Fix: If WAL is corrupt and Prometheus fails to start:
           # WARNING: This deletes uncompacted samples — back up the data directory first if possible.
           rm -rf /prometheus/wal
           # Prometheus will create a new WAL on startup

  -> If startup is slow but WAL size is reasonable
     -> Check total number of blocks:
        ls -la /prometheus/ | grep -c "^d"
        -> If hundreds of blocks exist
           -> Compaction is behind.
              Fix: Let Prometheus run; it will compact blocks automatically.
              Fix: Reduce retention to limit total block count:
                --storage.tsdb.retention.time=7d

  -> If Prometheus crashes during WAL replay
     -> Check for disk space issues:
        df -h /prometheus
        -> If disk is full
           -> Free disk space or expand volume.
           -> Reduce retention.
           -> Delete old blocks manually if necessary:
              # List blocks sorted by time
              ls -lt /prometheus/
              # Remove oldest block directories (format: 01BKGV7JBM69T2G1BGBGM6KB12/)
```

---

## 6. Configuration Errors

### Validate configuration file

```bash
promtool check config /etc/prometheus/prometheus.yml
```

Expected output on success:

```
Checking /etc/prometheus/prometheus.yml
 SUCCESS: /etc/prometheus/prometheus.yml is valid prometheus config file syntax
```

Expected output on failure:

```
Checking /etc/prometheus/prometheus.yml
 FAILED: parsing YAML file /etc/prometheus/prometheus.yml: yaml: line 15: did not find expected key
```

### Validate rule files

```bash
promtool check rules /etc/prometheus/rules/*.yml
```

Expected output on success:

```
Checking /etc/prometheus/rules/alerts.yml
 SUCCESS: 5 rules found
```

Expected output on failure:

```
Checking /etc/prometheus/rules/alerts.yml
 FAILED: group "example", rule 3, "MyAlert": could not parse expression: 1:14: parse error: unexpected end of input
```

### Decision tree for config issues

```
If Prometheus fails to start or reload fails
  -> Run promtool check config:
     promtool check config /etc/prometheus/prometheus.yml
     -> If FAILED
        -> Fix the YAML syntax or config error at the reported line number.
     -> If SUCCESS
        -> Run promtool check rules on all rule files:
           promtool check rules /etc/prometheus/rules/*.yml
           -> If FAILED
              -> Fix the PromQL expression or rule syntax at the reported location.
           -> If SUCCESS
              -> Check Prometheus logs for the specific error:
                 docker logs <container> 2>&1 | tail -50
                 -> If "error loading config" appears
                    -> File permissions or mount issue. Verify:
                       ls -la /etc/prometheus/prometheus.yml
                       # File must be readable by the prometheus user (typically uid 65534)
                 -> If reload returns 200 but config does not change
                    -> Check the loaded config via API:
                       curl -s http://localhost:9090/api/v1/status/config | python3 -m json.tool
                    -> Compare with the file on disk. If different, the reload
                       may have silently failed. Check logs for warnings.
```

### Using promtool inside a container

```bash
# If promtool is in the Prometheus image:
docker exec <container> promtool check config /etc/prometheus/prometheus.yml

# If using the official prom/prometheus image, promtool is at /bin/promtool:
docker exec <container> /bin/promtool check config /etc/prometheus/prometheus.yml
```

---

## 7. Useful Debug Queries

Diagnostic PromQL queries for monitoring Prometheus itself:

| Query | Purpose |
|---|---|
| `up` | Shows which targets are reachable (1) or unreachable (0) |
| `up == 0` | Lists only targets that are currently down |
| `scrape_duration_seconds` | Time taken to scrape each target (seconds) |
| `scrape_samples_scraped` | Number of samples scraped per target per scrape |
| `scrape_samples_post_metric_relabeling` | Samples remaining after metric_relabel_configs |
| `scrape_series_added` | New series added per scrape (churn indicator) |
| `prometheus_tsdb_head_series` | Total number of active time series in the head block |
| `prometheus_tsdb_head_chunks` | Total number of chunks in the head block |
| `prometheus_tsdb_head_chunks_storage_size_bytes` | Memory used by head chunks |
| `prometheus_tsdb_head_samples_appended_total` | Total samples appended (use `rate()`) |
| `rate(prometheus_tsdb_head_series_created_total[5m])` | Rate of new series creation |
| `rate(prometheus_tsdb_head_series_removed_total[5m])` | Rate of series removal (expiration) |
| `prometheus_tsdb_compactions_total` | Total TSDB compactions completed |
| `prometheus_tsdb_compaction_duration_seconds` | Duration of compactions |
| `prometheus_tsdb_wal_fsync_duration_seconds` | WAL fsync latency (use `histogram_quantile()`) |
| `prometheus_tsdb_blocks_loaded` | Number of TSDB blocks currently loaded |
| `prometheus_tsdb_storage_blocks_bytes` | Total bytes of all loaded blocks |
| `prometheus_config_last_reload_successful` | Whether the last config reload succeeded (1) or failed (0) |
| `prometheus_config_last_reload_success_timestamp_seconds` | Timestamp of last successful reload |
| `prometheus_rule_evaluation_duration_seconds` | Rule evaluation latency |
| `prometheus_rule_evaluation_failures_total` | Failed rule evaluations (use `rate()`) |
| `process_resident_memory_bytes` | Prometheus process RSS memory |
| `process_cpu_seconds_total` | Prometheus process CPU usage (use `rate()`) |
| `prometheus_http_request_duration_seconds_bucket` | HTTP API request latency histogram |

### Quick health dashboard queries

Check if last config reload succeeded:

```promql
prometheus_config_last_reload_successful
```

Find targets with slow scrapes (over 5 seconds):

```promql
scrape_duration_seconds > 5
```

Find targets dropping samples via relabeling:

```promql
scrape_samples_scraped - scrape_samples_post_metric_relabeling > 0
```

Monitor ingestion rate (samples per second):

```promql
rate(prometheus_tsdb_head_samples_appended_total[5m])
```

Check WAL fsync p99 latency:

```promql
histogram_quantile(0.99, rate(prometheus_tsdb_wal_fsync_duration_seconds_bucket[5m]))
```

---

## References

- [Management API](https://prometheus.io/docs/prometheus/latest/management_api/)
- [Querying API](https://prometheus.io/docs/prometheus/latest/querying/api/)
- [Instrumentation Best Practices](https://prometheus.io/docs/practices/instrumentation/)
- [TSDB Format](https://prometheus.io/docs/prometheus/latest/storage/)
