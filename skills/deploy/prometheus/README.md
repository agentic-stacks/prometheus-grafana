# Deploy Prometheus

## Prerequisites

### System Requirements

- **CPU**: 2+ cores recommended for production workloads
- **Memory**: 2 GB minimum; Prometheus memory usage scales with the number of active time series, ingestion rate, and query complexity. Plan approximately 1-2 KB per active time series.
- **Disk**: SSD strongly recommended. Storage usage is approximately 1-2 bytes per sample. Default retention is 15 days; size accordingly.
- **OS**: Linux (amd64, arm64), macOS (amd64, arm64), Windows (amd64); Linux is the primary production target.

### Ports

| Port | Protocol | Purpose |
|------|----------|---------|
| 9090 | TCP | Prometheus web UI, API, and metrics endpoint |

Ensure port 9090 (or your chosen `--web.listen-address`) is open in your firewall and not occupied by another process.

### Software

- **Binary install**: `tar`, `gzip`, `curl` or `wget`
- **Docker install**: Docker Engine 20.10+ or Docker Desktop
- **Helm install**: Kubernetes 1.25+, Helm 3.x, `kubectl` configured

---

## Install via Binary

### Download

Prometheus publishes pre-compiled binaries for every release on GitHub. Replace the version variable with your desired version.

**Prometheus v3.x (latest stable):**

```bash
PROM_VERSION="3.10.0"
curl -LO "https://github.com/prometheus/prometheus/releases/download/v${PROM_VERSION}/prometheus-${PROM_VERSION}.linux-amd64.tar.gz"
tar xvfz "prometheus-${PROM_VERSION}.linux-amd64.tar.gz"
cd "prometheus-${PROM_VERSION}.linux-amd64"
```

**Prometheus v2.x (legacy):**

```bash
PROM_VERSION="2.54.1"
curl -LO "https://github.com/prometheus/prometheus/releases/download/v${PROM_VERSION}/prometheus-${PROM_VERSION}.linux-amd64.tar.gz"
tar xvfz "prometheus-${PROM_VERSION}.linux-amd64.tar.gz"
cd "prometheus-${PROM_VERSION}.linux-amd64"
```

Replace `linux-amd64` with `linux-arm64`, `darwin-amd64`, or `darwin-arm64` for other platforms. Check https://prometheus.io/download/ for all available builds.

### Configure

Create a minimal `prometheus.yml` that scrapes Prometheus itself:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
```

Place this file at `/etc/prometheus/prometheus.yml` or in the same directory as the binary.

### Install as a Systemd Service

Copy the binaries and configuration into standard locations:

```bash
sudo useradd --no-create-home --shell /bin/false prometheus
sudo mkdir -p /etc/prometheus /var/lib/prometheus
sudo chown prometheus:prometheus /etc/prometheus /var/lib/prometheus

sudo cp prometheus promtool /usr/local/bin/
sudo chown prometheus:prometheus /usr/local/bin/prometheus /usr/local/bin/promtool

sudo cp prometheus.yml /etc/prometheus/prometheus.yml
sudo cp -r consoles console_libraries /etc/prometheus/
sudo chown -R prometheus:prometheus /etc/prometheus
```

Create the systemd unit file at `/etc/systemd/system/prometheus.service`:

```ini
[Unit]
Description=Prometheus Monitoring System
Documentation=https://prometheus.io/docs/introduction/overview/
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --web.listen-address=0.0.0.0:9090 \
  --web.enable-lifecycle
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

Enable and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus
sudo systemctl status prometheus
```

### Verify

```bash
curl -s http://localhost:9090/-/healthy
```

Expected response:

```
Prometheus Server is Healthy.
```

You can also open `http://localhost:9090` in a browser to access the web UI.

---

## Install via Docker

Docker images are published to Docker Hub as `prom/prometheus` and to Quay.io as `prometheus/prometheus`.

### Docker Run

```bash
docker run -d \
  --name prometheus \
  -p 9090:9090 \
  -v /path/to/prometheus.yml:/etc/prometheus/prometheus.yml \
  -v prometheus-data:/prometheus \
  prom/prometheus:v3.10.0
```

Replace `/path/to/prometheus.yml` with the absolute path to your configuration file. The named volume `prometheus-data` persists metric data across container restarts.

To use Prometheus v2.x, change the tag:

```bash
docker run -d \
  --name prometheus \
  -p 9090:9090 \
  -v /path/to/prometheus.yml:/etc/prometheus/prometheus.yml \
  -v prometheus-data:/prometheus \
  prom/prometheus:v2.54.1
```

### Docker Compose

Create a `docker-compose.yml` file:

```yaml
services:
  prometheus:
    image: prom/prometheus:v3.10.0
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.path=/prometheus"
      - "--storage.tsdb.retention.time=15d"
      - "--web.enable-lifecycle"
    restart: unless-stopped

volumes:
  prometheus-data:
    driver: local
```

Place your `prometheus.yml` in the same directory, then start:

```bash
docker compose up -d
```

### Verify

```bash
curl -s http://localhost:9090/-/healthy
```

Expected response:

```
Prometheus Server is Healthy.
```

---

## Install via Helm (Kubernetes)

The `prometheus-community` Helm charts are the standard method for deploying Prometheus on Kubernetes.

### Add the Helm Repository

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

### Install Prometheus

**Standalone Prometheus server:**

```bash
helm install prometheus prometheus-community/prometheus \
  --namespace monitoring \
  --create-namespace
```

**Full kube-prometheus-stack (includes Grafana, Alertmanager, and node-exporter):**

```bash
helm install kube-prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace
```

To customize values, create a `values.yaml` and pass it:

```bash
helm install prometheus prometheus-community/prometheus \
  --namespace monitoring \
  --create-namespace \
  -f values.yaml
```

### Verify

```bash
kubectl get pods -n monitoring
```

All pods should reach `Running` status. To access the Prometheus UI:

```bash
kubectl port-forward -n monitoring svc/prometheus-server 9090:80
```

Then open `http://localhost:9090` in your browser, or verify from the command line:

```bash
curl -s http://localhost:9090/-/healthy
```

For the kube-prometheus-stack, the service name differs:

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-kube-prome-prometheus 9090:9090
```

---

## Configuration

### Minimal prometheus.yml

A complete, valid configuration file that scrapes only Prometheus itself:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
```

- `scrape_interval`: How often Prometheus scrapes targets. Default is `1m`; `15s` is a common production value.
- `evaluation_interval`: How often Prometheus evaluates recording and alerting rules. Default is `1m`.

### Add Scrape Targets

To scrape Node Exporter running on the same host (default port 9100), add a job to `scrape_configs`:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node_exporter"
    static_configs:
      - targets: ["localhost:9100"]
```

To scrape multiple Node Exporter instances across hosts:

```yaml
  - job_name: "node_exporter"
    static_configs:
      - targets:
          - "192.168.1.10:9100"
          - "192.168.1.11:9100"
          - "192.168.1.12:9100"
```

### Reload Configuration

Prometheus can reload its configuration at runtime without restarting, using either of these methods:

**HTTP reload (requires `--web.enable-lifecycle` flag):**

```bash
curl -X POST http://localhost:9090/-/reload
```

**Signal-based reload:**

```bash
kill -HUP $(pidof prometheus)
```

Both methods reload `prometheus.yml` and all referenced rule files. If the new configuration is invalid, Prometheus logs an error and continues with the previous configuration.

You can validate configuration before applying:

```bash
promtool check config /etc/prometheus/prometheus.yml
```

---

## Version Notes

### Key Differences Between Prometheus v3.x and v2.x

| Area | v3.x | v2.x |
|------|------|------|
| **UI** | New React-based UI at `/query` | Classic UI at `/graph` |
| **UTF-8 support** | Full UTF-8 metric and label names enabled by default | Limited; UTF-8 support available behind feature flags in late 2.x |
| **Native histograms** | Stable and enabled by default | Experimental; requires `--enable-feature=native-histograms` |
| **OTLP ingestion** | Built-in OpenTelemetry OTLP receiver enabled by default | Requires `--enable-feature=otlp-write-receiver` in v2.47+ |
| **Remote write** | Remote write 2.0 protocol by default | Remote write 1.0 by default |
| **Minimum Go version** | Go 1.22+ | Go 1.21+ |
| **API stability** | Some v1 API endpoints removed or changed | Full v1 API |
| **Configuration** | Some deprecated config fields removed | All legacy fields available |

When upgrading from v2.x to v3.x, review the [Prometheus 3.0 migration guide](https://prometheus.io/docs/prometheus/latest/migration/) for breaking changes. The on-disk storage format is compatible; v3.x reads v2.x data without migration.

---

## Command-Line Flags Reference

| Flag | Default | Description |
|------|---------|-------------|
| `--config.file` | `prometheus.yml` | Path to the Prometheus configuration file. |
| `--storage.tsdb.path` | `data/` | Base path for metrics storage. |
| `--storage.tsdb.retention.time` | `15d` | How long to retain samples in storage. Supports units: `y`, `w`, `d`, `h`, `m`, `s`, `ms`. |
| `--web.listen-address` | `0.0.0.0:9090` | Address and port for the web UI, API, and telemetry. |
| `--web.enable-lifecycle` | `false` | Enable the `/-/reload` and `/-/quit` HTTP endpoints. Required for HTTP-based configuration reload. |
| `--web.enable-remote-write-receiver` | `false` | Enable the remote write receiver endpoint (`/api/v1/write`). |
| `--log.level` | `info` | Log verbosity. One of: `debug`, `info`, `warn`, `error`. |
| `--log.format` | `logfmt` | Log output format. One of: `logfmt`, `json`. |

View all available flags:

```bash
prometheus --help
```

---

## Sources

- https://prometheus.io/docs/prometheus/latest/installation/
- https://prometheus.io/docs/prometheus/latest/getting_started/
- https://prometheus.io/docs/prometheus/latest/configuration/configuration/
- https://prometheus.io/download/
