# Deploy Exporters

## What Are Exporters

Exporters are standalone processes that collect metrics from third-party systems and expose them in Prometheus format at an HTTP endpoint (typically `/metrics`). Prometheus scrapes these endpoints on a configured interval, converting the target system's native telemetry into the Prometheus data model.

The exporter pattern decouples metric collection from metric storage. Each exporter is purpose-built for one system (a database, a load balancer, hardware sensors, etc.), translating its internal statistics into Prometheus metric types (counters, gauges, histograms, summaries). This means you never modify the target system itself -- you deploy an exporter alongside it and point Prometheus at the exporter's `/metrics` endpoint.

Official exporters are maintained in the [Prometheus GitHub organization](https://github.com/prometheus). Community exporters follow the same conventions but are maintained independently.

---

## node_exporter (Host Metrics)

`node_exporter` exposes hardware and OS-level metrics for Linux and other Unix systems (CPU, memory, disk, network, filesystem, etc.). It is the most commonly deployed exporter and the starting point for host-level monitoring.

- **Default port**: 9100
- **Metrics endpoint**: `http://<host>:9100/metrics`
- **Repository**: <https://github.com/prometheus/node_exporter>

### Install via Binary

Download the pre-compiled binary from GitHub Releases and run it directly.

```bash
NODE_EXPORTER_VERSION="1.10.2"
curl -LO "https://github.com/prometheus/node_exporter/releases/download/v${NODE_EXPORTER_VERSION}/node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64.tar.gz"
tar xvfz "node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64.tar.gz"
cd "node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64"
./node_exporter
```

You should see output containing:

```
msg="Listening on" address=:9100
```

### Install via Docker

When running in Docker, host volume mounts are required so node_exporter can read host-level filesystems and proc data instead of the container's own.

```bash
docker run -d \
  --name node_exporter \
  --net host \
  --pid host \
  -v /proc:/host/proc:ro \
  -v /sys:/host/sys:ro \
  -v /:/rootfs:ro \
  prom/node-exporter:v1.10.2 \
  --path.procfs=/host/proc \
  --path.sysfs=/host/sys \
  --path.rootfs=/rootfs
```

> **Note**: `--net host` is strongly recommended so node_exporter reports the host's network metrics, not the container bridge.

### Run as systemd Service

Create a dedicated system user and a unit file for production deployments.

```bash
sudo useradd --no-create-home --shell /bin/false node_exporter
```

Copy the binary into place:

```bash
sudo cp node_exporter /usr/local/bin/
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
```

Create the unit file at `/etc/systemd/system/node_exporter.service`:

```ini
[Unit]
Description=Prometheus Node Exporter
Documentation=https://github.com/prometheus/node_exporter
After=network-online.target
Wants=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
sudo systemctl status node_exporter
```

### Key Collectors

The following table lists the most commonly referenced collectors. For the full list, see the [node_exporter README](https://github.com/prometheus/node_exporter#collectors).

| Collector | Metrics Exposed | Enabled by Default |
|-----------|----------------|--------------------|
| `cpu` | Per-CPU time in each mode (user, system, idle, iowait, etc.) | Yes |
| `meminfo` | Memory statistics from `/proc/meminfo` (total, free, buffers, cached, swap) | Yes |
| `diskstats` | Disk I/O statistics from `/proc/diskstats` (reads, writes, bytes, time) | Yes |
| `filesystem` | Filesystem usage (size, free, available, mount points) | Yes |
| `netdev` | Network interface statistics (bytes, packets, errors, drops) | Yes |
| `netstat` | Network statistics from `/proc/net/netstat` (TCP, UDP, IP counters) | Yes |
| `loadavg` | System load averages (1m, 5m, 15m) | Yes |
| `uname` | System information (kernel version, machine, OS) | Yes |
| `time` | System clock drift | Yes |
| `pressure` | Pressure Stall Information (PSI) for CPU, memory, I/O | Yes |
| `hwmon` | Hardware sensors (temperatures, fan speeds, voltages) | Yes |
| `textfile` | Custom metrics from `.prom` files in a directory | Yes |
| `systemd` | systemd unit states and counts | No |
| `processes` | Process counts by state (running, sleeping, zombie) | No |
| `ethtool` | NIC driver and device statistics | No |
| `tcpstat` | TCP connection statistics from `/proc/net/tcp` | No |
| `buddyinfo` | Memory fragmentation from `/proc/buddyinfo` | No |
| `interrupts` | Detailed interrupt counters from `/proc/interrupts` | No |
| `softirqs` | Softirq counts from `/proc/softirqs` | No |
| `perf` | Hardware and software perf counters | No |

### Useful Flags

| Flag | Description | Default |
|------|-------------|---------|
| `--web.listen-address` | Address and port to listen on | `:9100` |
| `--web.telemetry-path` | Path under which to expose metrics | `/metrics` |
| `--collector.textfile.directory` | Directory to read `.prom` files from for the textfile collector | (none) |
| `--no-collector.<name>` | Disable a specific default collector (e.g. `--no-collector.hwmon`) | - |
| `--collector.<name>` | Enable a specific disabled collector (e.g. `--collector.systemd`) | - |
| `--collector.disable-defaults` | Disable all default collectors; combine with `--collector.<name>` to enable only specific ones | `false` |
| `--log.level` | Log verbosity level | `info` |
| `--log.format` | Log output format (`logfmt` or `json`) | `logfmt` |

Example -- run with only CPU and memory collectors:

```bash
./node_exporter \
  --collector.disable-defaults \
  --collector.cpu \
  --collector.meminfo
```

Example -- enable the systemd collector in addition to all defaults:

```bash
./node_exporter --collector.systemd
```

### Add to Prometheus

Add a scrape job to `prometheus.yml`:

```yaml
scrape_configs:
  - job_name: "node"
    static_configs:
      - targets:
          - "node-host-1:9100"
          - "node-host-2:9100"
```

For service discovery, see the [Prometheus service discovery documentation](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#scrape_config).

### Verify

Confirm that node_exporter is exposing metrics:

```bash
curl -s http://localhost:9100/metrics | head -20
```

Check for a specific metric:

```bash
curl -s http://localhost:9100/metrics | grep "node_cpu_seconds_total"
```

Confirm Prometheus is scraping successfully by visiting `http://<prometheus-host>:9090/targets` and verifying the `node` job shows state **UP**.

---

## Common Exporters Reference

The following exporters cover the most frequently monitored infrastructure components.

| Exporter | Purpose | Default Port | Docs |
|----------|---------|-------------|------|
| [blackbox_exporter](https://github.com/prometheus/blackbox_exporter) | Probe endpoints over HTTP, HTTPS, DNS, TCP, ICMP | 9115 | [GitHub README](https://github.com/prometheus/blackbox_exporter/blob/master/README.md) |
| [mysqld_exporter](https://github.com/prometheus/mysqld_exporter) | MySQL/MariaDB server metrics | 9104 | [GitHub README](https://github.com/prometheus/mysqld_exporter/blob/main/README.md) |
| [postgres_exporter](https://github.com/prometheus-community/postgres_exporter) | PostgreSQL server metrics | 9187 | [GitHub README](https://github.com/prometheus-community/postgres_exporter/blob/master/README.md) |
| [redis_exporter](https://github.com/oliver006/redis_exporter) | Redis server metrics | 9121 | [GitHub README](https://github.com/oliver006/redis_exporter/blob/master/README.md) |
| [snmp_exporter](https://github.com/prometheus/snmp_exporter) | SNMP device metrics (routers, switches, appliances) | 9116 | [GitHub README](https://github.com/prometheus/snmp_exporter/blob/main/README.md) |
| [cAdvisor](https://github.com/google/cadvisor) | Container resource usage and performance (Docker, Kubernetes) | 8080 | [GitHub README](https://github.com/google/cadvisor/blob/master/README.md) |

### Quick Start Examples

**blackbox_exporter** -- HTTP probe:

```bash
docker run -d \
  --name blackbox_exporter \
  -p 9115:9115 \
  prom/blackbox-exporter:v0.25.0
```

Prometheus scrape config for HTTP probing:

```yaml
scrape_configs:
  - job_name: "blackbox-http"
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - "https://example.com"
          - "https://prometheus.io"
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: "blackbox-exporter:9115"
```

**mysqld_exporter**:

```bash
docker run -d \
  --name mysqld_exporter \
  -p 9104:9104 \
  -e DATA_SOURCE_NAME="exporter:password@(mysql-host:3306)/" \
  prom/mysqld-exporter:v0.16.0
```

**postgres_exporter**:

```bash
docker run -d \
  --name postgres_exporter \
  -p 9187:9187 \
  -e DATA_SOURCE_NAME="postgresql://exporter:password@postgres-host:5432/postgres?sslmode=disable" \
  prometheuscommunity/postgres-exporter:v0.16.0
```

**redis_exporter**:

```bash
docker run -d \
  --name redis_exporter \
  -p 9121:9121 \
  oliver006/redis_exporter:v1.67.0 \
  --redis.addr=redis://redis-host:6379
```

**cAdvisor**:

```bash
docker run -d \
  --name cadvisor \
  -p 8080:8080 \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:ro \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  --volume=/dev/disk/:/dev/disk:ro \
  --privileged \
  --device=/dev/kmsg \
  gcr.io/cadvisor/cadvisor:v0.51.0
```

For a comprehensive list of all available exporters, see the [official exporters documentation](https://prometheus.io/docs/instrumenting/exporters/).

---

## Pushgateway

### When to Use

The [Pushgateway](https://github.com/prometheus/pushgateway) acts as a metrics cache for short-lived and batch jobs that may terminate before Prometheus can scrape them. Instead of exposing a `/metrics` endpoint and waiting to be scraped, these jobs push their metrics to the Pushgateway, which then exposes them for Prometheus to collect.

Use the Pushgateway when:

- A batch job runs for seconds or minutes and then exits
- A cron job completes before the next Prometheus scrape interval
- A serverless function or ephemeral container produces metrics during execution

Do **not** use the Pushgateway for:

- Long-running services (use a normal exporter or client library instead)
- Aggregating metrics from multiple instances (the Pushgateway is a cache, not an aggregator)
- Proxying metrics from machines behind a firewall (use federation or remote write)

### Install

**Binary**:

```bash
PUSHGATEWAY_VERSION="1.11.0"
curl -LO "https://github.com/prometheus/pushgateway/releases/download/v${PUSHGATEWAY_VERSION}/pushgateway-${PUSHGATEWAY_VERSION}.linux-amd64.tar.gz"
tar xvfz "pushgateway-${PUSHGATEWAY_VERSION}.linux-amd64.tar.gz"
cd "pushgateway-${PUSHGATEWAY_VERSION}.linux-amd64"
./pushgateway
```

**Docker**:

```bash
docker run -d \
  --name pushgateway \
  -p 9091:9091 \
  prom/pushgateway:v1.11.0
```

The Pushgateway UI is available at `http://localhost:9091`.

### Push Metrics Examples

Push a single metric for a job named `my_batch_job`:

```bash
echo "batch_job_duration_seconds 42.7" | curl --data-binary @- \
  http://localhost:9091/metrics/job/my_batch_job
```

Push a metric with additional grouping labels:

```bash
echo "batch_job_records_processed 15832" | curl --data-binary @- \
  http://localhost:9091/metrics/job/my_batch_job/instance/host1
```

Push multiple metrics at once:

```bash
cat <<'EOF' | curl --data-binary @- http://localhost:9091/metrics/job/my_batch_job/instance/host1
# TYPE batch_job_duration_seconds gauge
batch_job_duration_seconds 42.7
# TYPE batch_job_records_processed counter
batch_job_records_processed 15832
# TYPE batch_job_success gauge
batch_job_success 1
EOF
```

Delete metrics for a specific job and instance:

```bash
curl -X DELETE http://localhost:9091/metrics/job/my_batch_job/instance/host1
```

### Scrape Config

Add the Pushgateway as a scrape target in `prometheus.yml`. The `honor_labels: true` setting is critical -- it tells Prometheus to preserve the `job` and `instance` labels pushed by the batch job rather than overwriting them with the Pushgateway's own labels.

```yaml
scrape_configs:
  - job_name: "pushgateway"
    honor_labels: true
    static_configs:
      - targets:
          - "pushgateway-host:9091"
```

Without `honor_labels: true`, every metric scraped from the Pushgateway would have `job="pushgateway"` and `instance="pushgateway-host:9091"`, losing the original job identity pushed by the batch process. With it enabled, the pushed `job` label (e.g. `my_batch_job`) is preserved, and the Pushgateway's own labels are available as `exported_job` and `exported_instance` if there is a conflict.

### Verify

Confirm the Pushgateway is running:

```bash
curl -s http://localhost:9091/metrics | head -10
```

After pushing a metric, verify it appears:

```bash
curl -s http://localhost:9091/metrics | grep "batch_job_duration_seconds"
```

Check the Pushgateway status page in Prometheus at `http://<prometheus-host>:9090/targets` and verify the `pushgateway` job shows state **UP**.
