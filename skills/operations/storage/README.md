# Storage

## Local Storage (TSDB)

Prometheus stores time-series data locally using an embedded TSDB (time-series database). The on-disk layout under `--storage.tsdb.path` (default: `data/`) is:

```
data/
├── chunks_head/        # Current in-memory block's memory-mapped chunks
│   └── 000001
├── wal/                # Write-ahead log segments (128 MB each)
│   ├── 00000000
│   ├── 00000001
│   └── 00000002
├── 01BKGV7JBM69T2G1... # Completed block directories (2-hour blocks)
│   ├── chunks/
│   │   └── 000001      # Chunk segment files (up to 512 MB each)
│   ├── index           # Maps metric names and labels to time series
│   ├── meta.json       # Block metadata (time range, stats)
│   └── tombstones      # Deletion markers
├── 01BKGTZQ1...
│   └── ...
└── lock                # Process lock file
```

**Blocks:** Incoming samples are first kept in an in-memory head block spanning two hours. When the head block is full, it is flushed (compacted) to disk as an immutable two-hour block.

**WAL (Write-Ahead Log):** The current in-memory head block is secured against crashes by a write-ahead log stored in the `wal/` directory. WAL files are written in 128 MB segments. Prometheus retains a minimum of three WAL files at any time. High-traffic servers retain more to preserve at least two hours of raw data.

**Compaction:** A background process merges smaller two-hour blocks into larger blocks. Compacted blocks can span up to 10% of the retention time, or 31 days, whichever is smaller. Compaction typically completes within two hours.

## Retention Configuration

### By Time

```
--storage.tsdb.retention.time=15d
```

| Flag | Default | Description |
|------|---------|-------------|
| `--storage.tsdb.retention.time` | `15d` | How long to keep data. Supported units: `y`, `w`, `d`, `h`, `m`, `s`, `ms`. |

### By Size

```
--storage.tsdb.retention.size=512MB
```

| Flag | Default | Description |
|------|---------|-------------|
| `--storage.tsdb.retention.size` | `0` (disabled) | Maximum total size of all blocks. Units: `B`, `KB`, `MB`, `GB`, `TB`, `PB`, `EB` (powers of 2). |

When both flags are set, whichever policy triggers first causes data removal.

> **Tip:** Set `--storage.tsdb.retention.size` to at most 80-85% of your allocated Prometheus disk space to allow headroom for the WAL and `chunks_head/` directory.

## Disk Space Planning

Prometheus stores an average of 1-2 bytes per sample. Use this formula to estimate required disk space:

```
needed_disk_space = retention_time_seconds * ingested_samples_per_second * bytes_per_sample
```

**Example:** 10,000 samples/sec, 1.5 bytes/sample, 15-day retention:

```
15 * 24 * 3600 * 10000 * 1.5 = ~19.4 GB
```

To find your current ingestion rate:

```promql
rate(prometheus_tsdb_head_samples_appended_total[5m])
```

## Remote Write

Remote write sends samples from the local TSDB to a remote endpoint in real time. Configure it in `prometheus.yml`:

```yaml
remote_write:
  # The URL of the endpoint to send samples to.
  - url: <string>

    # Protobuf message format.
    # prometheus.WriteRequest = Remote Write 1.0 (default, will be deprecated)
    # io.prometheus.write.v2.Request = Remote Write 2.0 (improved efficiency, sends metadata)
    [ protobuf_message: <prometheus.WriteRequest | io.prometheus.write.v2.Request> | default = prometheus.WriteRequest ]

    # Timeout for requests to the remote write endpoint.
    [ remote_timeout: <duration> | default = 30s ]

    # Custom HTTP headers to be sent along with each remote write request.
    headers:
      [ <string>: <string> ... ]

    # List of remote write relabel configurations.
    # Applied after external labels. Can limit which samples are sent.
    write_relabel_configs:
      [ - <relabel_config> ... ]

    # Name of the remote write config (must be unique if specified).
    [ name: <string> ]

    # Enables sending of exemplars over remote write.
    [ send_exemplars: <boolean> | default = false ]

    # Enables sending of native histograms over remote write.
    [ send_native_histograms: <boolean> | default = false ]

    # Queue configuration for tuning throughput and buffering.
    queue_config:
      # Number of samples to buffer per shard before blocking reads from the WAL.
      [ capacity: <int> | default = 10000 ]
      # Maximum number of shards (concurrency).
      [ max_shards: <int> | default = 50 ]
      # Minimum number of shards (concurrency).
      [ min_shards: <int> | default = 1 ]
      # Maximum number of samples per send.
      [ max_samples_per_send: <int> | default = 2000 ]
      # Maximum time a sample will wait for a send.
      [ batch_send_deadline: <duration> | default = 5s ]
      # Initial retry delay. Doubled for every retry.
      [ min_backoff: <duration> | default = 30ms ]
      # Maximum retry delay.
      [ max_backoff: <duration> | default = 5s ]
      # Retry upon receiving a 429 status code from remote-write storage.
      [ retry_on_http_429: <boolean> | default = false ]
      # Samples older than this are not sent. 0s means all samples are sent.
      [ sample_age_limit: <duration> | default = 0s ]

    # Configures sending of series metadata to remote storage.
    # Only applies when using prometheus.WriteRequest message.
    # With io.prometheus.write.v2.Request, metadata is always sent.
    metadata_config:
      # Whether metric metadata is sent to remote storage.
      [ send: <boolean> | default = true ]
      # How frequently metric metadata is sent to remote storage.
      [ send_interval: <duration> | default = 1m ]
      # Maximum number of samples per metadata send.
      [ max_samples_per_send: <int> | default = 500 ]
```

**Example** -- send all metrics to a Mimir endpoint:

```yaml
remote_write:
  - url: "https://mimir.example.com/api/v1/push"
    remote_timeout: 30s
    queue_config:
      capacity: 10000
      max_shards: 50
      batch_send_deadline: 5s
    headers:
      X-Scope-OrgID: "my-tenant"
```

## Remote Read

Remote read lets Prometheus query an external data store for historical data. Configure it in `prometheus.yml`:

```yaml
remote_read:
  # The URL of the endpoint to query from.
  - url: <string>

    # Name of the remote read config (must be unique if specified).
    [ name: <string> ]

    # An optional list of equality matchers which must be present in a
    # selector to query this remote read endpoint.
    required_matchers:
      [ <labelname>: <labelvalue> ... ]

    # Timeout for requests to the remote read endpoint.
    [ remote_timeout: <duration> | default = 1m ]

    # Custom HTTP headers to be sent along with each remote read request.
    headers:
      [ <string>: <string> ... ]

    # Whether reads should be made for queries for time ranges that
    # the local storage should have complete data for.
    [ read_recent: <boolean> | default = false ]

    # Whether to use external labels as selectors for the remote read endpoint.
    [ filter_external_labels: <boolean> | default = true ]
```

**Example** -- read from a long-term Thanos store with required matchers:

```yaml
remote_read:
  - url: "https://thanos-query.example.com/api/v1/read"
    remote_timeout: 2m
    read_recent: false
    required_matchers:
      cluster: "production"
```

## Federation

Federation lets one Prometheus server scrape selected time-series data from another Prometheus server's `/federate` endpoint.

### Hierarchical Federation

Hierarchical federation scales Prometheus across large environments (tens of data centers, millions of nodes). Lower-level servers collect detailed instance-level metrics, while higher-level servers aggregate job-level data from subordinates.

```yaml
scrape_configs:
  - job_name: 'federate'
    scrape_interval: 15s

    honor_labels: true
    metrics_path: '/federate'

    params:
      'match[]':
        - '{job="prometheus"}'
        - '{__name__=~"job:.*"}'

    static_configs:
      - targets:
        - 'source-prometheus-1:9090'
        - 'source-prometheus-2:9090'
        - 'source-prometheus-3:9090'
```

Key points:
- `honor_labels: true` preserves the original labels from the source server and avoids `exported_` prefixing.
- `match[]` parameters control which series are pulled. Use recording rules (e.g., `job:*`) at the lower level to pre-aggregate before federating.

### Cross-Service Federation

Cross-service federation pulls selected metrics from one service's Prometheus into another service's Prometheus. This enables alerting and querying across both datasets from a single server.

**Example:** A cluster scheduler exposes resource usage metrics (CPU, memory) while an application exposes business metrics. Federation lets the application's Prometheus pull cluster resource data for unified alerting.

```yaml
scrape_configs:
  - job_name: 'cluster-resource-metrics'
    scrape_interval: 15s

    honor_labels: true
    metrics_path: '/federate'

    params:
      'match[]':
        - '{job="kubelet"}'
        - '{__name__=~"node_.*"}'

    static_configs:
      - targets:
        - 'cluster-prometheus:9090'
```

## Long-Term Storage Options

Prometheus local TSDB is designed for short-to-medium retention. For long-term storage, use a dedicated system with `remote_write`:

| Solution | Type | Notes |
|----------|------|-------|
| **Thanos** | Sidecar + Object Store | Deploys as a sidecar to Prometheus. Uploads blocks to object storage (S3, GCS, Azure). Global query view across multiple Prometheus instances. |
| **Grafana Mimir** | Remote Write Target | Horizontally scalable, multi-tenant. Accepts `remote_write`. Compatible with PromQL. Successor to Cortex. |
| **Cortex** | Remote Write Target | Horizontally scalable, multi-tenant TSDB. Original project; Mimir is its successor. Still maintained. |
| **VictoriaMetrics** | Remote Write Target | High-performance, cost-effective. Single-node and cluster editions. Accepts `remote_write` natively. |

## WAL and Crash Recovery

### How WAL Works

The write-ahead log ensures durability for the in-memory head block:

1. Every incoming sample is first written to the WAL before being added to the in-memory head block.
2. WAL files are stored in the `wal/` subdirectory as 128 MB segments.
3. Prometheus retains a minimum of three WAL files at any time.

### Replay on Startup

When Prometheus starts (or restarts after a crash), it replays the WAL to reconstruct the in-memory head block. The replay time depends on the amount of WAL data -- typically proportional to the number of active time series and the ingestion rate.

### WAL Compression

```
--storage.tsdb.wal-compression
```

| Flag | Default | Description |
|------|---------|-------------|
| `--storage.tsdb.wal-compression` | `true` (since v2.20.0) | Enables Snappy compression for WAL segments. Reduces WAL size by roughly half with minimal CPU overhead. |

WAL compression was introduced in Prometheus v2.11.0 as an opt-in feature and became the default in v2.20.0.

## Version Notes

### Prometheus v3.x Storage Changes

Prometheus 3.0 introduced the following storage-related changes:

- **Native histograms in TSDB:** Full support for native (sparse) histograms in storage. Native histograms are stored more efficiently than classic histograms, reducing the number of individual time series.
- **UTF-8 metric and label names:** The TSDB supports UTF-8 encoded metric and label names, removing the previous restriction to `[a-zA-Z_:][a-zA-Z0-9_:]*`.
- **Remote Write 2.0 (`io.prometheus.write.v2.Request`):** A new wire protocol that sends metadata, created timestamps, and native histograms by default with improved efficiency. Configurable via the `protobuf_message` field in `remote_write`.
- **Out-of-order ingestion enabled by default:** The `--storage.tsdb.out-of-order-time-window` flag defaults to 10 minutes (was `0s` in v2.x), allowing late-arriving samples to be ingested.
- **WAL compression on by default:** `--storage.tsdb.wal-compression` is enabled by default (inherited from v2.20.0+).
- **Created timestamps:** Prometheus 3.x sends created timestamps to remote write endpoints by default, improving counter reset detection.

---

**Sources:**
- https://prometheus.io/docs/prometheus/latest/storage/
- https://prometheus.io/docs/prometheus/latest/federation/
- https://prometheus.io/docs/prometheus/latest/configuration/configuration/
