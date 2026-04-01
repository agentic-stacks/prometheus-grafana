# Prometheus v3 Known Issues

Breaking changes, known bugs, deprecations, and migration steps for Prometheus 3.x releases.

Sources: [Prometheus v3 releases](https://github.com/prometheus/prometheus/releases), [v3 migration guide](https://prometheus.io/docs/prometheus/latest/migration/), [Prometheus blog](https://prometheus.io/blog/).

---

## v3.0.0 Breaking Changes

### Scrape protocol strictness rejects targets with missing Content-Type
**Symptom:** Scrape targets that previously worked now fail with `INVALID` status and "non-compliant Content-Type header" errors in logs.
**Cause:** Prometheus v3 no longer silently falls back to Prometheus text format when the Content-Type header is missing or invalid. v2 accepted anything.
**Workaround:** Add `fallback_scrape_protocol: PrometheusText0.0.4` to affected scrape configs:
```yaml
scrape_configs:
  - job_name: legacy-target
    fallback_scrape_protocol: PrometheusText0.0.4
```
**Affected versions:** 3.0.0 through latest
**Status:** By design; permanent behavior change

### Remote-write HTTP/2 disabled by default
**Symptom:** Remote-write throughput drops or connections behave differently after upgrade. Endpoints expecting HTTP/2 may reject connections.
**Cause:** Default for `http_config.enable_http2` in remote-write changed from `true` to `false`.
**Workaround:** Explicitly re-enable in remote-write config:
```yaml
remote_write:
  - url: https://remote-endpoint/api/v1/write
    http_config:
      enable_http2: true
```
**Affected versions:** 3.0.0 through latest
**Status:** By design; permanent behavior change

### PromQL regex `.` now matches newline characters
**Symptom:** Label matchers using `.` in regex return unexpected results that include newlines in multi-line label values.
**Cause:** Prometheus v3 changed regex semantics so `.` matches all characters including `\n`, for performance.
**Workaround:** Replace `.` with `[^\n]` in regex matchers where newline matching is not desired:
```promql
# Before (v2 behavior)
metric{label=~"foo.bar"}
# After (v3 compatible)
metric{label=~"foo[^\n]bar"}
```
**Affected versions:** 3.0.0 through latest
**Status:** By design; permanent behavior change

### Range selectors are now left-open (exclude boundary sample)
**Symptom:** Queries return fewer samples than in v2. Subqueries and recording rules may produce slightly different results or gaps.
**Cause:** Range selectors changed from left-closed/right-closed `[start, end]` to left-open/right-closed `(start, end]`. A sample exactly at the lower boundary is now excluded.
**Workaround:** Extend range windows by one scrape interval to compensate. For example, change `foo[1m:1m]` to `foo[2m:1m]`.
**Affected versions:** 3.0.0 through latest
**Status:** By design; permanent behavior change

### `holt_winters` renamed to `double_exponential_smoothing`
**Symptom:** PromQL queries using `holt_winters()` fail with "unknown function" error.
**Cause:** Function renamed for clarity. Additionally moved behind experimental feature flag.
**Workaround:** Rename function calls and enable the feature flag:
```bash
--enable-feature=promql-experimental-functions
```
```promql
# Before
holt_winters(metric[10m], 0.5, 0.5)
# After
double_exponential_smoothing(metric[10m], 0.5, 0.5)
```
**Affected versions:** 3.0.0 through latest
**Status:** By design; permanent rename

### `scrape_classic_histograms` config renamed
**Symptom:** Prometheus fails to start or logs warnings about unknown config field `scrape_classic_histograms`.
**Cause:** Config parameter renamed to `always_scrape_classic_histograms`.
**Workaround:** Find and replace in all config files:
```yaml
# Before
scrape_classic_histograms: true
# After
always_scrape_classic_histograms: true
```
**Affected versions:** 3.0.0 through latest
**Status:** By design; permanent rename

### Alertmanager v1 API no longer supported
**Symptom:** Prometheus fails to send alerts. Logs show API errors when communicating with Alertmanager.
**Cause:** Alertmanager v1 API support was removed. Prometheus v3 requires Alertmanager 0.16.0+ with v2 API.
**Workaround:** Upgrade Alertmanager to 0.16.0 or later. Update config:
```yaml
alerting:
  alertmanagers:
    - api_version: v2
```
**Affected versions:** 3.0.0 through latest
**Status:** By design; v1 API permanently removed

### Kubernetes v1beta1 EndpointSlice and Ingress SD removed
**Symptom:** Kubernetes service discovery fails to discover targets. Logs show errors about unsupported API versions.
**Cause:** Removed support for `discovery.k8s.io/v1beta1` EndpointSlice (deprecated since K8s v1.25) and `networking.k8s.io/v1beta1` Ingress (deprecated since K8s v1.22).
**Workaround:** Ensure Kubernetes cluster runs v1.25+ and uses the stable v1 API versions. No config change needed if cluster is current.
**Affected versions:** 3.0.0 through latest
**Status:** By design; permanent removal

### Removed deprecated CLI flags cause startup failure
**Symptom:** Prometheus fails to start with "unknown flag" errors for `--storage.tsdb.allow-overlapping-blocks`, `--alertmanager.timeout`, `--storage.tsdb.retention`, or `--enable-feature` with removed feature flags.
**Cause:** Deprecated flags and feature flags removed: `remote-write-receiver`, `promql-at-modifier`, `promql-negative-offset`, `expand-external-labels`, `no-default-scrape-port`, `agent`.
**Workaround:** Remove deprecated flags from startup scripts, systemd units, and Kubernetes manifests. Use replacement flags:
- `--enable-feature=remote-write-receiver` becomes `--web.enable-remote-write-receiver`
- `--enable-feature=agent` becomes `--agent`
- `--storage.tsdb.retention` becomes `--storage.tsdb.retention.time`
**Affected versions:** 3.0.0 through latest
**Status:** By design; permanent removal

### `le` and `quantile` label values normalized to float format
**Symptom:** Histogram and summary queries using exact label matches on `le` or `quantile` fail. For example `{le="1"}` no longer matches because the stored value is now `1.0`.
**Cause:** Prometheus v3 normalizes `le` and `quantile` label values upon ingestion to consistent float representation.
**Workaround:** Update all queries, recording rules, and alert expressions to use float format:
```promql
# Before
histogram_bucket{le="1"}
# After
histogram_bucket{le="1.0"}
```
**Affected versions:** 3.0.0 through latest
**Status:** By design; permanent behavior change

### New UI missing exemplar display and heatmaps
**Symptom:** Exemplars and heatmap visualizations are absent from the Prometheus web UI after upgrade.
**Cause:** The completely redesigned PromLens-style UI does not yet support these features.
**Workaround:** Use `--enable-feature=old-ui` flag to revert to the classic UI temporarily.
**Affected versions:** 3.0.0 through 3.8.x (old-ui flag availability)
**Status:** Open; features being ported to new UI

### GOMAXPROCS and GOMEMLIMIT auto-set from container limits
**Symptom:** Changed CPU and memory behavior in containerized environments. Prometheus may use more or less CPU than expected.
**Cause:** Prometheus v3 automatically sets `GOMAXPROCS` to match Linux CPU quota and `GOMEMLIMIT` to match container memory limit.
**Workaround:** Disable with flags if the automatic tuning causes issues:
```bash
--no-auto-gomaxprocs
--no-auto-gomemlimit
```
**Affected versions:** 3.0.0 through latest
**Status:** By design; can be disabled

### External labels expand environment variables by default
**Symptom:** External label values containing `$` characters are unexpectedly interpreted as environment variable references and replaced (or become empty strings if undefined).
**Cause:** The `expand-external-labels` feature flag is now default behavior.
**Workaround:** Escape literal `$` characters with `$$`:
```yaml
global:
  external_labels:
    cost_center: "dept$$100"  # produces "dept$100"
```
**Affected versions:** 3.0.0 through latest
**Status:** By design; permanent behavior change

### Log format changed breaks log parsers
**Symptom:** Log parsing pipelines (Loki, Fluentd, etc.) stop extracting fields from Prometheus logs.
**Cause:** Logging switched from `go-kit/log` to `log/slog`. Field names changed: `ts=` became `time=`, `caller=` became `source=`.
**Workaround:** Update log parsing configurations to match new format:
```
# Old format
ts=2024-01-01T00:00:00Z caller=main.go:100 level=info msg="..."
# New format
time=2024-01-01T00:00:00Z source=main.go:100 level=info msg="..."
```
**Affected versions:** 3.0.0 through latest
**Status:** By design; permanent format change

### UTF-8 metric and label names enabled by default
**Symptom:** Metrics with UTF-8 characters in names are accepted where they were previously rejected. Downstream systems that do not support UTF-8 may break.
**Cause:** UTF-8 support is now on by default; the `utf8-name` feature flag was removed.
**Workaround:** To restore v2 behavior, set in config:
```yaml
global:
  metric_name_validation_scheme: legacy
```
**Affected versions:** 3.0.0 through latest
**Status:** By design; can be reverted via config

---

## v3.8.0

### Remote-write 2.0 protocol changes
**Symptom:** Remote-write receivers that implemented earlier remote-write 2.0 draft specs reject writes from Prometheus 3.8+.
**Cause:** Remote-write 2.0 updated to rc.4 specification with protocol changes.
**Workaround:** Ensure remote-write receivers support the rc.4 spec, or fall back to remote-write 1.0.
**Affected versions:** 3.8.0 through latest
**Status:** Open until remote-write 2.0 is finalized

### New "unknown" alerting state
**Symptom:** Alert rules show an "unknown" state in the UI and API that was not present before. Alerting dashboards or automation may not handle this state.
**Cause:** New state added for rules that have not been evaluated yet.
**Workaround:** Update alerting dashboards and automation to recognize the `unknown` state alongside `inactive`, `pending`, and `firing`.
**Affected versions:** 3.8.0 through latest
**Status:** By design; permanent addition

---

## v3.9.0

### Native histograms feature flag becomes no-op
**Symptom:** Setting `--enable-feature=native-histograms` no longer enables native histogram scraping. Native histograms stop being collected.
**Cause:** The feature flag was deprecated; native histograms are now controlled solely via config.
**Workaround:** Add `scrape_native_histograms: true` to scrape configs that need native histograms:
```yaml
scrape_configs:
  - job_name: my-target
    scrape_native_histograms: true
```
**Affected versions:** 3.9.0 through latest
**Status:** By design; permanent change

---

## v3.10.0

### PromQL `fill()` modifier changes empty series behavior
**Symptom:** Queries using the new `fill()`, `fill_left()`, or `fill_right()` binary operation modifiers may produce unexpected results if operators assume empty series remain empty after binops.
**Cause:** New modifiers allow filling missing values in binary operations, changing default empty-series semantics.
**Workaround:** Review and test any queries using these new modifiers. They are opt-in and do not affect existing queries.
**Affected versions:** 3.10.0 through latest
**Status:** By design; new feature

---

## v3.11.0-rc.0

### Promtool debug output moved to stderr
**Symptom:** Scripts that parse promtool stdout for debug information stop working.
**Cause:** Debug output redirected from stdout to stderr for cleaner tool output.
**Workaround:** Update scripts to read debug output from stderr:
```bash
# Before
promtool check config prometheus.yml 2>/dev/null | parse_debug
# After
promtool check config prometheus.yml 2>&1 1>/dev/null | parse_debug
```
**Affected versions:** 3.11.0-rc.0 through latest
**Status:** By design; permanent change

### Hetzner SD datacenter label deprecation
**Symptom:** Deprecation warnings in logs about Hetzner service discovery datacenter labels.
**Cause:** Datacenter label format is being changed with a July 2026 deadline for migration.
**Workaround:** Update relabeling rules to use new label format before July 2026.
**Affected versions:** 3.11.0-rc.0 through latest
**Status:** Open; deadline July 2026

---

## TSDB Compatibility Note

Prometheus v3 writes a TSDB format that is only backward-compatible with v2.55+. If you need to downgrade from v3 to v2, you must use v2.55.0 or later to read the data directory.
