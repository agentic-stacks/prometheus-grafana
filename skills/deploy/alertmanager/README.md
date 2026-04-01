# Deploy Alertmanager

## Prerequisites

### System Requirements

- **CPU**: 1+ cores; Alertmanager is lightweight compared to Prometheus.
- **Memory**: 512 MB minimum for moderate alert volumes. Memory scales with the number of active alerts and silences.
- **Disk**: Minimal. Alertmanager stores notification state and silences in a local file (`data/`). SSD is not required.
- **OS**: Linux (amd64, arm64), macOS (amd64, arm64), Windows (amd64); Linux is the primary production target.

### Ports

| Port | Protocol | Purpose |
|------|----------|---------|
| 9093 | TCP | Alertmanager web UI, API, and webhook receiver |
| 9094 | TCP/UDP | Cluster gossip communication (high availability) |

Ensure ports 9093 and 9094 (if clustering) are open in your firewall and not occupied by another process.

### Software

- **Binary install**: `tar`, `gzip`, `curl` or `wget`
- **Docker install**: Docker Engine 20.10+ or Docker Desktop
- **amtool**: Included in the Alertmanager release tarball; used for configuration validation and alert management

---

## Install via Binary

### Download

Alertmanager publishes pre-compiled binaries for every release on GitHub. Replace the version variable with your desired version.

```bash
AM_VERSION="0.28.1"
curl -LO "https://github.com/prometheus/alertmanager/releases/download/v${AM_VERSION}/alertmanager-${AM_VERSION}.linux-amd64.tar.gz"
tar xvfz "alertmanager-${AM_VERSION}.linux-amd64.tar.gz"
cd "alertmanager-${AM_VERSION}.linux-amd64"
```

Replace `linux-amd64` with `linux-arm64`, `darwin-amd64`, or `darwin-arm64` for other platforms. Check https://prometheus.io/download/#alertmanager for all available builds.

### Configure

Create a minimal `alertmanager.yml` (see the [Configure alertmanager.yml](#configure-alertmanageryml) section below for the full configuration reference).

### Install as a Systemd Service

Copy the binaries and configuration into standard locations:

```bash
sudo useradd --no-create-home --shell /bin/false alertmanager
sudo mkdir -p /etc/alertmanager /var/lib/alertmanager
sudo chown alertmanager:alertmanager /etc/alertmanager /var/lib/alertmanager

sudo cp alertmanager amtool /usr/local/bin/
sudo chown alertmanager:alertmanager /usr/local/bin/alertmanager /usr/local/bin/amtool

sudo cp alertmanager.yml /etc/alertmanager/alertmanager.yml
sudo chown -R alertmanager:alertmanager /etc/alertmanager
```

Create the systemd unit file at `/etc/systemd/system/alertmanager.service`:

```ini
[Unit]
Description=Prometheus Alertmanager
Documentation=https://prometheus.io/docs/alerting/latest/alertmanager/
Wants=network-online.target
After=network-online.target

[Service]
User=alertmanager
Group=alertmanager
Type=simple
ExecStart=/usr/local/bin/alertmanager \
  --config.file=/etc/alertmanager/alertmanager.yml \
  --storage.path=/var/lib/alertmanager \
  --web.listen-address=0.0.0.0:9093
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

Enable and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable alertmanager
sudo systemctl start alertmanager
sudo systemctl status alertmanager
```

---

## Install via Docker

Docker images are published to Docker Hub as `prom/alertmanager` and to Quay.io as `prometheus/alertmanager`.

### Docker Run

```bash
docker run -d \
  --name alertmanager \
  -p 9093:9093 \
  -v /path/to/alertmanager.yml:/etc/alertmanager/alertmanager.yml \
  -v alertmanager-data:/alertmanager \
  prom/alertmanager:v0.28.1
```

Replace `/path/to/alertmanager.yml` with the absolute path to your configuration file. The named volume `alertmanager-data` persists notification state and silences across container restarts.

### Docker Compose

Create a `docker-compose.yml` file:

```yaml
services:
  alertmanager:
    image: prom/alertmanager:v0.28.1
    container_name: alertmanager
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
      - alertmanager-data:/alertmanager
    command:
      - "--config.file=/etc/alertmanager/alertmanager.yml"
      - "--storage.path=/alertmanager"
    restart: unless-stopped

volumes:
  alertmanager-data:
    driver: local
```

Place your `alertmanager.yml` in the same directory, then start:

```bash
docker compose up -d
```

---

## Configure alertmanager.yml

### Minimal Configuration (Email Receiver)

A complete, valid configuration file with an email receiver:

```yaml
global:
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.example.com:587'
  smtp_from: 'alertmanager@example.com'
  smtp_auth_username: 'alertmanager@example.com'
  smtp_auth_password: 'secret'
  smtp_require_tls: true

route:
  receiver: 'email-notifications'
  group_by: ['alertname', 'cluster']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h

receivers:
  - name: 'email-notifications'
    email_configs:
      - to: 'oncall@example.com'
        send_resolved: true
```

### Route Tree

The route tree controls how alerts are grouped, throttled, and routed to receivers. Routes are matched top-down; the first matching route wins unless `continue: true` is set.

```yaml
route:
  receiver: 'default'
  group_by: ['alertname', 'cluster']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h

  routes:
    # Critical alerts go to PagerDuty immediately
    - matchers:
        - severity="critical"
      receiver: 'pagerduty-critical'
      group_wait: 10s
      continue: false

    # Warning alerts matching a specific team go to that team's Slack
    - matchers:
        - severity="warning"
        - team=~"^(platform|infra)$"
      receiver: 'slack-platform'
      group_by: ['alertname', 'service']
      continue: false

    # All database alerts go to the DBA channel regardless of severity
    - matchers:
        - service=~"mysql|postgres|redis"
      receiver: 'slack-dba'
      continue: true

    # Catch-all for everything else
    - matchers:
        - severity=~"warning|info"
      receiver: 'email-notifications'
```

**Key route fields:**

| Field | Description |
|-------|-------------|
| `receiver` | Name of the receiver to handle matched alerts. |
| `group_by` | Labels used to aggregate alerts into groups. Use `['...']` to disable grouping (send each alert individually). |
| `group_wait` | How long to wait before sending the first notification for a new group. Default: `30s`. |
| `group_interval` | How long to wait before sending updates for an existing group when new alerts arrive. Default: `5m`. |
| `repeat_interval` | How long to wait before re-sending a notification for an alert that is still firing. Default: `4h`. |
| `matchers` | List of matchers using label matching operators: `=` (equality), `!=` (inequality), `=~` (regex match), `!~` (regex non-match). |
| `continue` | If `true`, continue evaluating sibling routes after this match. Default: `false`. |

### Receivers

#### Email

```yaml
receivers:
  - name: 'email-notifications'
    email_configs:
      - to: 'oncall@example.com'
        from: 'alertmanager@example.com'
        smarthost: 'smtp.example.com:587'
        auth_username: 'alertmanager@example.com'
        auth_password: 'secret'
        send_resolved: true
        headers:
          Subject: '[{{ .Status | toUpper }}] {{ .GroupLabels.alertname }}'
```

When `smtp_smarthost`, `smtp_from`, `smtp_auth_username`, and `smtp_auth_password` are set in the `global` section, only `to` is required in individual `email_configs`.

#### Slack

```yaml
receivers:
  - name: 'slack-platform'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX'
        channel: '#alerts-platform'
        title: '{{ .GroupLabels.alertname }}'
        text: >-
          {{ range .Alerts }}
          *Alert:* {{ .Annotations.summary }}
          *Severity:* {{ .Labels.severity }}
          *Description:* {{ .Annotations.description }}
          {{ end }}
        send_resolved: true
```

Alternatively, set `slack_api_url` in the `global` section and omit `api_url` from individual configs.

#### PagerDuty

```yaml
receivers:
  - name: 'pagerduty-critical'
    pagerduty_configs:
      - routing_key: 'your-pagerduty-integration-key'
        severity: '{{ if eq .GroupLabels.severity "critical" }}critical{{ else }}warning{{ end }}'
        description: '{{ .GroupLabels.alertname }}: {{ .CommonAnnotations.summary }}'
        send_resolved: true
```

The `routing_key` is the integration key from a PagerDuty Events v2 integration. Alternatively, use `service_key` for the v1 API. The default PagerDuty URL (`https://events.pagerduty.com/v2/enqueue`) can be overridden with `pagerduty_url` in the `global` section.

#### Webhook

```yaml
receivers:
  - name: 'webhook-receiver'
    webhook_configs:
      - url: 'http://example.com/webhook'
        send_resolved: true
        max_alerts: 0
```

Alertmanager sends a POST request with a JSON payload containing alert details. Set `max_alerts` to `0` to include all alerts in a single request (default behavior). The webhook endpoint must respond with a 2xx status code.

### Inhibition Rules

Inhibition rules suppress notifications for certain alerts when other alerts are already firing. This prevents alert storms. The `equal` field lists labels that must have the same values on both source and target alerts for the inhibition to apply.

```yaml
inhibit_rules:
  # When a critical alert fires, inhibit warning alerts for the same alertname and cluster
  - source_matchers:
      - severity="critical"
    target_matchers:
      - severity="warning"
    equal: ['alertname', 'cluster']

  # When a cluster is unreachable, inhibit all alerts from that cluster
  - source_matchers:
      - alertname="ClusterUnreachable"
    target_matchers:
      - severity=~"warning|info"
    equal: ['cluster']
```

### Complete Configuration Example

```yaml
global:
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.example.com:587'
  smtp_from: 'alertmanager@example.com'
  smtp_auth_username: 'alertmanager@example.com'
  smtp_auth_password: 'secret'
  smtp_require_tls: true
  slack_api_url: 'https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX'

route:
  receiver: 'email-notifications'
  group_by: ['alertname', 'cluster']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h

  routes:
    - matchers:
        - severity="critical"
      receiver: 'pagerduty-critical'
      group_wait: 10s

    - matchers:
        - severity="warning"
      receiver: 'slack-platform'

receivers:
  - name: 'email-notifications'
    email_configs:
      - to: 'oncall@example.com'
        send_resolved: true

  - name: 'pagerduty-critical'
    pagerduty_configs:
      - routing_key: 'your-pagerduty-integration-key'
        send_resolved: true

  - name: 'slack-platform'
    slack_configs:
      - channel: '#alerts-platform'
        send_resolved: true

inhibit_rules:
  - source_matchers:
      - severity="critical"
    target_matchers:
      - severity="warning"
    equal: ['alertname', 'cluster']
```

---

## Connect Prometheus to Alertmanager

Add the `alerting` section to your `prometheus.yml` to tell Prometheus where to send firing alerts:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - 'localhost:9093'

rule_files:
  - 'rules/*.yml'

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
```

For multiple Alertmanager instances (high availability), list all peers. Prometheus sends alerts to all configured Alertmanagers; do not place a load balancer between Prometheus and Alertmanager:

```yaml
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - 'alertmanager-1:9093'
            - 'alertmanager-2:9093'
            - 'alertmanager-3:9093'
```

After editing `prometheus.yml`, reload the Prometheus configuration:

```bash
curl -X POST http://localhost:9090/-/reload
```

---

## High Availability

Alertmanager supports a high-availability mode using a gossip protocol for peer communication over port 9094 (TCP and UDP). In HA mode, multiple Alertmanager instances form a cluster and deduplicate notifications so that only one notification is sent per alert group.

### Start a Cluster

Start each Alertmanager instance with the `--cluster.peer` flag pointing to at least one other peer:

**Node 1 (seed):**

```bash
alertmanager \
  --config.file=/etc/alertmanager/alertmanager.yml \
  --storage.path=/var/lib/alertmanager \
  --web.listen-address=0.0.0.0:9093 \
  --cluster.listen-address=0.0.0.0:9094
```

**Node 2:**

```bash
alertmanager \
  --config.file=/etc/alertmanager/alertmanager.yml \
  --storage.path=/var/lib/alertmanager \
  --web.listen-address=0.0.0.0:9093 \
  --cluster.listen-address=0.0.0.0:9094 \
  --cluster.peer=alertmanager-1:9094
```

**Node 3:**

```bash
alertmanager \
  --config.file=/etc/alertmanager/alertmanager.yml \
  --storage.path=/var/lib/alertmanager \
  --web.listen-address=0.0.0.0:9093 \
  --cluster.listen-address=0.0.0.0:9094 \
  --cluster.peer=alertmanager-1:9094
```

Each additional node only needs to reference one existing peer; the gossip protocol propagates membership to all nodes. All instances must use the same `alertmanager.yml` configuration.

### Docker Compose HA Cluster

```yaml
services:
  alertmanager-1:
    image: prom/alertmanager:v0.28.1
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
    command:
      - "--config.file=/etc/alertmanager/alertmanager.yml"
      - "--storage.path=/alertmanager"
      - "--cluster.listen-address=0.0.0.0:9094"

  alertmanager-2:
    image: prom/alertmanager:v0.28.1
    ports:
      - "9094:9093"
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
    command:
      - "--config.file=/etc/alertmanager/alertmanager.yml"
      - "--storage.path=/alertmanager"
      - "--cluster.listen-address=0.0.0.0:9094"
      - "--cluster.peer=alertmanager-1:9094"

  alertmanager-3:
    image: prom/alertmanager:v0.28.1
    ports:
      - "9095:9093"
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
    command:
      - "--config.file=/etc/alertmanager/alertmanager.yml"
      - "--storage.path=/alertmanager"
      - "--cluster.listen-address=0.0.0.0:9094"
      - "--cluster.peer=alertmanager-1:9094"
```

### Verify Cluster Status

Check the cluster status via the API:

```bash
curl -s http://localhost:9093/api/v2/status | python3 -m json.tool
```

The response includes a `cluster` object with `peers` listing all connected nodes.

---

## Verify

### Validate Configuration

Use `amtool` to check the configuration file for syntax errors before starting or reloading Alertmanager:

```bash
amtool check-config /etc/alertmanager/alertmanager.yml
```

Expected output on success:

```
Checking '/etc/alertmanager/alertmanager.yml'  SUCCESS
Found:
 - global config
 - route
 - 1 inhibit rules
 - 3 receivers
 - 0 templates
```

### Health Check

```bash
curl -s http://localhost:9093/-/healthy
```

Expected response:

```
OK
```

### Readiness Check

```bash
curl -s http://localhost:9093/-/ready
```

Expected response:

```
OK
```

### Send a Test Alert

Post a test alert to the Alertmanager API to verify end-to-end notification delivery:

```bash
curl -X POST http://localhost:9093/api/v2/alerts \
  -H "Content-Type: application/json" \
  -d '[
    {
      "labels": {
        "alertname": "TestAlert",
        "severity": "warning",
        "cluster": "test-cluster",
        "service": "test-service"
      },
      "annotations": {
        "summary": "This is a test alert",
        "description": "Verifying Alertmanager notification pipeline"
      },
      "startsAt": "'"$(date -u +%Y-%m-%dT%H:%M:%S.000Z)"'",
      "generatorURL": "http://localhost:9090/graph"
    }
  ]'
```

Verify the alert is active:

```bash
curl -s http://localhost:9093/api/v2/alerts | python3 -m json.tool
```

### View Alerts with amtool

```bash
amtool alert --alertmanager.url=http://localhost:9093
```

---

## Command-Line Flags Reference

| Flag | Default | Description |
|------|---------|-------------|
| `--config.file` | `alertmanager.yml` | Path to the Alertmanager configuration file. |
| `--storage.path` | `data/` | Base path for notification state and silences storage. |
| `--web.listen-address` | `0.0.0.0:9093` | Address and port for the web UI and API. |
| `--web.external-url` | | The URL under which Alertmanager is externally reachable (used for links in notifications). |
| `--web.route-prefix` | | Prefix for the internal routes of the web endpoints. Defaults to the path from `--web.external-url`. |
| `--web.config.file` | | Path to the web configuration file (TLS and basic auth). Experimental. |
| `--cluster.listen-address` | `0.0.0.0:9094` | Address and port for cluster (gossip) communication. Set to empty string to disable HA mode. |
| `--cluster.peer` | | Initial peer for cluster gossip (may be repeated for multiple peers). |
| `--cluster.peer-timeout` | `15s` | Timeout for establishing peer connections. |
| `--cluster.gossip-interval` | `200ms` | Interval between gossip state syncs. |
| `--cluster.pushpull-interval` | `1m0s` | Interval for full state syncs between peers. |
| `--cluster.settle-timeout` | `1m0s` | Maximum time to wait for cluster connections before sending notifications. |
| `--cluster.reconnect-interval` | `10s` | Interval between reconnect attempts to lost peers. |
| `--cluster.reconnect-timeout` | `6h0m0s` | Duration to attempt reconnects before removing a peer. |
| `--cluster.tls-config` | | Path to the mutual TLS configuration file for cluster gossip traffic. |
| `--data.retention` | `120h` | How long to keep data (notification state and silences). |
| `--alerts.gc-interval` | `30m` | Interval between garbage collection of expired alerts. |
| `--log.level` | `info` | Log verbosity. One of: `debug`, `info`, `warn`, `error`. |
| `--log.format` | `logfmt` | Log output format. One of: `logfmt`, `json`. |

View all available flags:

```bash
alertmanager --help
```

---

## Sources

- https://prometheus.io/docs/alerting/latest/alertmanager/
- https://prometheus.io/docs/alerting/latest/configuration/
- https://prometheus.io/docs/alerting/latest/https/
- https://prometheus.io/download/#alertmanager
