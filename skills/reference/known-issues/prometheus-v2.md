# Prometheus v2 Known Issues

Breaking changes, known bugs, deprecations, and migration steps for the final Prometheus 2.x releases (v2.53.0 through v2.55.1). These are the versions operators must be on before upgrading to v3.

Sources: [Prometheus v2.x releases](https://github.com/prometheus/prometheus/releases), [v3 migration guide](https://prometheus.io/docs/prometheus/latest/migration/).

---

## v2.53.0

### GOGC runtime default changed from 100 to 75
**Symptom:** Prometheus uses slightly more CPU but significantly less memory than before the upgrade. Garbage collection runs more frequently.
**Cause:** Go garbage collector target percentage (`GOGC`) changed from 100 to 75, causing more aggressive garbage collection with reduced heap usage.
**Workaround:** If the CPU increase is unacceptable, override at startup:
```bash
GOGC=100 prometheus --config.file=prometheus.yml
```
**Affected versions:** 2.53.0 through 2.55.1
**Status:** By design; carried forward into v3

### Query logger file descriptor leak
**Symptom:** Prometheus slowly leaks file descriptors when the query logger is enabled. Eventually hits OS file descriptor limits, causing query failures.
**Cause:** Bug in query logger not properly closing file handles.
**Workaround:** Fixed in v2.53.0. Upgrade from earlier versions if using the query logger.
**Affected versions:** 2.45.0 through 2.52.0
**Status:** Fixed in 2.53.0

### OTLP generates unnecessary `target_info` time series
**Symptom:** Unexpected `target_info` metric series appear when using OTLP ingestion, inflating active series count and storage.
**Cause:** Bug in OTLP translation layer that generated `target_info` even when not needed.
**Workaround:** Fixed in v2.53.0. Upgrade from earlier versions.
**Affected versions:** 2.47.0 through 2.52.0
**Status:** Fixed in 2.53.0

### Scraping silently accepts native histograms when feature is disabled
**Symptom:** Native histogram samples are ingested even though the `native-histograms` feature flag is not enabled, inflating storage usage.
**Cause:** Missing feature flag check in scrape ingestion path.
**Workaround:** Fixed in v2.53.0. Upgrade from earlier versions.
**Affected versions:** 2.50.0 through 2.52.0
**Status:** Fixed in 2.53.0

---

## v2.54.0 / v2.54.1

### Native histogram panics and incorrect results
**Symptom:** Prometheus panics during queries involving native histograms. Some histogram queries return incorrect results or trigger warnings.
**Cause:** Multiple bugs in native histogram query handling.
**Workaround:** Upgrade to v2.55.0 which includes comprehensive fixes. If on v2.54.x, avoid using native histograms in production queries.
**Affected versions:** 2.54.0 through 2.54.1
**Status:** Fixed in 2.55.0

### Exemplar drops during protobuf scraping
**Symptom:** Exemplars attached to metrics scraped via protobuf format are silently dropped and not stored.
**Cause:** Bug in protobuf scrape parsing that failed to extract exemplar data.
**Workaround:** Fixed in v2.55.0. Use text format scraping as interim workaround.
**Affected versions:** 2.54.0 through 2.54.1
**Status:** Fixed in 2.55.0

---

## v2.55.0

### `sort_by_label` produces unstable ordering
**Symptom:** Queries using `sort_by_label()` or `sort_by_label_desc()` return results in inconsistent order across evaluations when label values are identical.
**Cause:** Sort implementation did not use a stable sort algorithm.
**Workaround:** Fixed in v2.55.0. Upgrade from earlier versions.
**Affected versions:** 2.50.0 through 2.54.1
**Status:** Fixed in 2.55.0

### Remote-write V2 metadata sending issues
**Symptom:** Metadata (metric descriptions, types) not reliably sent to remote-write V2 receivers.
**Cause:** Bug in experimental remote-write V2 implementation.
**Workaround:** Fixed in v2.55.0. Upgrade from earlier versions, or use remote-write V1.
**Affected versions:** 2.54.0 through 2.54.1
**Status:** Fixed in 2.55.0

### Remote-write returns 5xx instead of 4xx for duplicate labels
**Symptom:** Remote-write receiver responds with 5xx (server error) when a time series contains duplicate labels. Senders retry indefinitely because 5xx implies a transient error.
**Cause:** Incorrect HTTP status code returned for a client error condition.
**Workaround:** Fixed in v2.55.0. Upgrade the receiver.
**Affected versions:** 2.47.0 through 2.54.1
**Status:** Fixed in 2.55.0

---

## v2.55.1

### `round()` function does not remove `__name__` label
**Symptom:** Queries using `round()` produce results that still carry the `__name__` label, which can cause unexpected behavior in aggregations and recording rules that assume the label is stripped by functions.
**Cause:** Bug in the `round()` function implementation not following the convention of other PromQL functions that modify values.
**Workaround:** Fixed in v2.55.1. Upgrade from v2.55.0.
**Affected versions:** 2.55.0
**Status:** Fixed in 2.55.1

---

## Deprecations in the v2.x Line (Relevant for v3 Migration)

These features are deprecated in late v2.x and removed in v3.0.0:

### `--enable-feature=promql-at-modifier`
**Symptom:** Deprecation warning in logs.
**Cause:** AT modifier is now standard PromQL; flag is no longer needed.
**Workaround:** Remove the flag from startup configuration before upgrading to v3.
**Affected versions:** Deprecated in 2.53.0+; removed in 3.0.0
**Status:** Removed in 3.0.0

### `--enable-feature=promql-negative-offset`
**Symptom:** Deprecation warning in logs.
**Cause:** Negative offset is now standard PromQL.
**Workaround:** Remove the flag from startup configuration before upgrading to v3.
**Affected versions:** Deprecated in 2.53.0+; removed in 3.0.0
**Status:** Removed in 3.0.0

### `--storage.tsdb.retention` flag
**Symptom:** Deprecation warning in logs.
**Cause:** Replaced by `--storage.tsdb.retention.time`.
**Workaround:** Switch to `--storage.tsdb.retention.time` before upgrading to v3.
**Affected versions:** Deprecated since 2.7.0; removed in 3.0.0
**Status:** Removed in 3.0.0

### `--storage.tsdb.allow-overlapping-blocks` flag
**Symptom:** Deprecation warning in logs.
**Cause:** Overlapping blocks are now always allowed.
**Workaround:** Remove the flag from startup configuration.
**Affected versions:** Deprecated in late 2.x; removed in 3.0.0
**Status:** Removed in 3.0.0

### `--alertmanager.timeout` flag
**Symptom:** Deprecation warning in logs.
**Cause:** Replaced by per-alertmanager timeout in config file.
**Workaround:** Move timeout configuration to the alerting config block.
**Affected versions:** Deprecated in late 2.x; removed in 3.0.0
**Status:** Removed in 3.0.0

---

## v3 Upgrade Readiness

Before upgrading from v2.x to v3.0, ensure you are on **v2.55.0 or later**. The v3 TSDB format is only backward-compatible with v2.55+, so downgrading to earlier v2 releases is not possible once v3 has written to the data directory.
