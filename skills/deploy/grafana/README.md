# Deploy Grafana

Install, configure, and provision Grafana for use with Prometheus.

Reference docs:
- https://grafana.com/docs/grafana/latest/setup-grafana/installation/
- https://grafana.com/docs/grafana/latest/setup-grafana/configure-grafana/
- https://grafana.com/docs/grafana/latest/datasources/prometheus/
- https://grafana.com/docs/grafana/latest/administration/provisioning/

---

## 1. Prerequisites

| Requirement | Minimum |
|---|---|
| CPU | 1 core |
| Memory | 512 MB |
| Disk | 1 GB |
| Database | SQLite 3 (built-in), PostgreSQL 12+, or MySQL 8.0+ |
| Browser | Chrome, Firefox, Safari, Edge (latest two major versions) |
| Network | Port 3000/tcp open for the Grafana web UI |

A running Prometheus instance is required to use Grafana as a metrics visualization layer. See `skills/deploy/prometheus/README.md`.

---

## 2. Install via Package Manager (Debian/Ubuntu)

### Install dependencies

```bash
sudo apt-get install -y apt-transport-https wget gnupg
```

### Import the GPG key

```bash
sudo mkdir -p /etc/apt/keyrings
sudo wget -O /etc/apt/keyrings/grafana.asc https://apt.grafana.com/gpg-full.key
sudo chmod 644 /etc/apt/keyrings/grafana.asc
```

### Add the stable repository

```bash
echo "deb [signed-by=/etc/apt/keyrings/grafana.asc] https://apt.grafana.com stable main" | \
  sudo tee /etc/apt/sources.list.d/grafana.list
```

### Install Grafana OSS

```bash
sudo apt-get update
sudo apt-get install -y grafana
```

Or install Grafana Enterprise (free, includes all OSS features):

```bash
sudo apt-get install -y grafana-enterprise
```

### Start and enable the service

```bash
sudo systemctl daemon-reload
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

### Verify

```bash
sudo systemctl status grafana-server
```

---

## 3. Install via Package Manager (RHEL/CentOS)

### Import the GPG key

```bash
wget -q -O gpg.key https://rpm.grafana.com/gpg.key
sudo rpm --import gpg.key
```

### Create the repository file

```bash
cat <<'EOF' | sudo tee /etc/yum.repos.d/grafana.repo
[grafana]
name=grafana
baseurl=https://rpm.grafana.com
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://rpm.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
EOF
```

### Install Grafana OSS

```bash
sudo dnf install -y grafana
```

Or install Grafana Enterprise:

```bash
sudo dnf install -y grafana-enterprise
```

### Start and enable the service

```bash
sudo systemctl daemon-reload
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

---

## 4. Install via Docker

### Quick start

```bash
docker run -d -p 3000:3000 --name=grafana grafana/grafana-enterprise:12.4.2
```

### With persistent storage (named volume)

```bash
docker volume create grafana-storage

docker run -d -p 3000:3000 --name=grafana \
  --volume grafana-storage:/var/lib/grafana \
  grafana/grafana-enterprise:12.4.2
```

### With bind mount

```bash
mkdir -p ./grafana-data

docker run -d -p 3000:3000 --name=grafana \
  --user "$(id -u)" \
  --volume "$PWD/grafana-data:/var/lib/grafana" \
  grafana/grafana-enterprise:12.4.2
```

### Docker Compose

Create `docker-compose.yaml`:

```yaml
services:
  prometheus:
    image: prom/prometheus:v3.10.0
    container_name: prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - prometheus-data:/prometheus
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro

  grafana:
    image: grafana/grafana-enterprise:12.4.2
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    volumes:
      - grafana-storage:/var/lib/grafana
      - ./provisioning:/etc/grafana/provisioning:ro
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
    depends_on:
      - prometheus

volumes:
  grafana-storage: {}
  prometheus-data: {}
```

Start the stack:

```bash
docker compose up -d
```

### Pass configuration via environment variables

Any `grafana.ini` setting can be overridden with environment variables using the pattern `GF_<SECTION>_<KEY>`. For example:

```bash
docker run -d -p 3000:3000 --name=grafana \
  -e "GF_SERVER_ROOT_URL=http://grafana.example.com:3000" \
  -e "GF_LOG_LEVEL=debug" \
  grafana/grafana-enterprise:12.4.2
```

---

## 5. Install via Helm (Kubernetes)

### Add the Helm repository

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

### Create a namespace

```bash
kubectl create namespace monitoring
```

### Install with defaults

```bash
helm install grafana grafana/grafana \
  --namespace monitoring
```

### Install with persistent storage and Prometheus data source

```bash
helm install grafana grafana/grafana \
  --namespace monitoring \
  --set persistence.enabled=true \
  --set persistence.size=5Gi \
  --set adminPassword=changeme \
  --set "datasources.datasources\\.yaml.apiVersion=1" \
  --set "datasources.datasources\\.yaml.datasources[0].name=Prometheus" \
  --set "datasources.datasources\\.yaml.datasources[0].type=prometheus" \
  --set "datasources.datasources\\.yaml.datasources[0].url=http://prometheus-server.monitoring.svc.cluster.local:9090" \
  --set "datasources.datasources\\.yaml.datasources[0].isDefault=true" \
  --set "datasources.datasources\\.yaml.datasources[0].access=proxy"
```

### Retrieve the admin password

```bash
kubectl get secret --namespace monitoring grafana \
  -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

### Port-forward to access the UI

```bash
kubectl port-forward --namespace monitoring svc/grafana 3000:80
```

### Upgrade or change values

```bash
helm upgrade grafana grafana/grafana \
  --namespace monitoring \
  -f values.yaml
```

### Verify

```bash
helm list -n monitoring
kubectl get all -n monitoring
```

---

## 6. Initial Setup

### Default credentials

| Field | Value |
|---|---|
| URL | `http://<host>:3000` |
| Username | `admin` |
| Password | `admin` |

On first login Grafana forces a password change. Set a strong password immediately.

### Add Prometheus data source (UI)

1. Log in to Grafana at `http://localhost:3000`.
2. Open the side menu and go to **Connections > Data sources**.
3. Click **Add data source**.
4. Select **Prometheus** from the list.
5. Set the **Prometheus server URL** to `http://localhost:9090` (or `http://prometheus:9090` inside Docker Compose, or `http://prometheus-server.monitoring.svc.cluster.local:9090` in Kubernetes).
6. Leave **Access** as `Server (default)`.
7. Scroll down and click **Save & test**.
8. A green banner reading "Successfully queried the Prometheus API" confirms the connection.

### Add Prometheus data source (API)

```bash
curl -X POST http://admin:admin@localhost:3000/api/datasources \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Prometheus",
    "type": "prometheus",
    "access": "proxy",
    "url": "http://localhost:9090",
    "isDefault": true,
    "jsonData": {
      "httpMethod": "POST",
      "prometheusType": "Prometheus"
    }
  }'
```

---

## 7. Configuration (grafana.ini)

The main configuration file lives at `/etc/grafana/grafana.ini` (Debian/RPM installs). In Docker, override settings with environment variables using the pattern `GF_<SECTION>_<KEY>` (all uppercase).

Restart Grafana after any changes to `grafana.ini`:

```bash
sudo systemctl restart grafana-server
```

### [server]

```ini
[server]
# Protocol (http, https, h2, socket)
protocol = http

# The IP address to bind to (empty = all interfaces)
http_addr =

# The HTTP port to bind to
http_port = 3000

# The public-facing domain name
domain = localhost

# Full URL used for redirects and OAuth callbacks
root_url = %(protocol)s://%(domain)s:%(http_port)s/

# Set to true if Grafana is behind a reverse proxy at a sub-path
serve_from_sub_path = false

# Enable gzip compression
enable_gzip = false
```

### [security]

```ini
[security]
# Default admin user created on first startup
admin_user = admin

# Default admin password (change on first login)
admin_password = admin

# Used for signing cookies and other internal values
secret_key = SW2YcwTIb9zpOOhoPsMm

# Disable gravatar profile images
disable_gravatar = false

# Prevent embedding in iframes from other origins
cookie_secure = false
cookie_samesite = lax

# Content Security Policy
content_security_policy = false
```

### [auth]

```ini
[auth]
# Set to true to disable login form
disable_login_form = false

# Set to true to disable signout menu
disable_signout_menu = false

# How long a login cookie lasts (in days)
login_cookie_name = grafana_session
login_maximum_inactive_lifetime_duration = 7d
login_maximum_lifetime_duration = 30d
```

### [database]

```ini
[database]
# Database type: sqlite3, mysql, postgres
type = sqlite3

# For sqlite3, the path to the database file
path = grafana.db

# For mysql/postgres:
# host = 127.0.0.1:3306
# name = grafana
# user = root
# password =

# Maximum number of open database connections
max_open_conn = 0

# SSL mode for PostgreSQL: disable, require, verify-ca, verify-full
# ssl_mode = disable
```

---

## 8. Provisioning Data Sources

Place provisioning YAML files in `/etc/grafana/provisioning/datasources/`. Grafana reads them on startup and creates or updates the data sources automatically.

### `/etc/grafana/provisioning/datasources/prometheus.yaml`

```yaml
apiVersion: 1

deleteDatasources: []

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    orgId: 1
    url: http://localhost:9090
    isDefault: true
    jsonData:
      httpMethod: POST
      prometheusType: Prometheus
      prometheusVersion: 3.2.1
      cacheLevel: Medium
      incrementalQuerying: true
      incrementalQueryOverlapWindow: 10m
    version: 1
    editable: false
```

### Environment variable substitution

Use `$VAR` or `${VAR}` in provisioning YAML to inject environment variables at startup:

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://${PROMETHEUS_HOST}:${PROMETHEUS_PORT}
    isDefault: true
    jsonData:
      httpMethod: POST
    editable: false
```

---

## 9. Provisioning Dashboards

Place dashboard provisioning YAML in `/etc/grafana/provisioning/dashboards/`. This tells Grafana where to find JSON dashboard files on disk.

### `/etc/grafana/provisioning/dashboards/default.yaml`

```yaml
apiVersion: 1

providers:
  - name: default
    orgId: 1
    folder: ""
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    allowUiUpdates: true
    options:
      path: /var/lib/grafana/dashboards
      foldersFromFilesStructure: true
```

### Example dashboard JSON

Place JSON dashboard files in `/var/lib/grafana/dashboards/`. Example Prometheus overview dashboard at `/var/lib/grafana/dashboards/prometheus-overview.json`:

```json
{
  "annotations": { "list": [] },
  "editable": true,
  "fiscalYearStartMonth": 0,
  "graphTooltip": 0,
  "links": [],
  "panels": [
    {
      "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
      "fieldConfig": {
        "defaults": { "unit": "short" },
        "overrides": []
      },
      "gridPos": { "h": 8, "w": 12, "x": 0, "y": 0 },
      "id": 1,
      "targets": [
        {
          "expr": "up",
          "legendFormat": "{{instance}}",
          "refId": "A"
        }
      ],
      "title": "Targets Up",
      "type": "timeseries"
    },
    {
      "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
      "fieldConfig": {
        "defaults": { "unit": "short" },
        "overrides": []
      },
      "gridPos": { "h": 8, "w": 12, "x": 12, "y": 0 },
      "id": 2,
      "targets": [
        {
          "expr": "prometheus_tsdb_head_series",
          "legendFormat": "{{instance}}",
          "refId": "A"
        }
      ],
      "title": "Head Series",
      "type": "timeseries"
    }
  ],
  "schemaVersion": 40,
  "tags": ["prometheus"],
  "templating": {
    "list": [
      {
        "current": {},
        "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
        "definition": "label_values(up, instance)",
        "name": "instance",
        "query": "label_values(up, instance)",
        "type": "query"
      }
    ]
  },
  "time": { "from": "now-1h", "to": "now" },
  "title": "Prometheus Overview",
  "uid": "prometheus-overview"
}
```

### Docker Compose volume mount for provisioning

Mount the local `provisioning/` directory into the container:

```yaml
services:
  grafana:
    image: grafana/grafana-enterprise:12.4.2
    volumes:
      - ./provisioning:/etc/grafana/provisioning:ro
      - ./dashboards:/var/lib/grafana/dashboards:ro
      - grafana-storage:/var/lib/grafana
```

---

## 10. Version Notes

### Grafana v12.x (current release line, v12.4.2)

Key changes from v11.x:

| Change | Details |
|---|---|
| Git Sync | Dashboards can be versioned in Git and edited via pull request workflows. |
| Drilldown apps GA | Metrics, Logs, and Traces Drilldown tools are now generally available. |
| New dashboard schema | Dashboard v2 schema introduced (migration is non-reversible). |
| Table panel refactor | Large tables load and filter significantly faster. |
| SQL Expressions (preview) | Cross-data-source data manipulation using SQL. |
| Angular removal | Angular plugin framework has been fully removed. Plugins must use React. |
| SCIM provisioning | User and team provisioning from identity providers via SCIM. |
| Alert "Recovering" state | New state reduces noise from flapping alerts. |
| `editors_can_admin` removed | The deprecated configuration option has been deleted. |
| Stricter data source UIDs | Data source UID format is now strictly validated. |
| Dynamic dashboards | Conditional logic and tab-based layouts within dashboards. |
| Migration Assistant GA | Tool for migrating self-managed Grafana to Grafana Cloud. |

### Upgrading from v11.x to v12.x

1. Back up the Grafana database and configuration before upgrading.
2. Check that all installed plugins are compatible with v12.x (Angular plugins will not load).
3. Review the [What's new in Grafana v12.0](https://grafana.com/docs/grafana/latest/whatsnew/whats-new-in-v12-0/) page for the complete list of breaking changes.
4. Test the upgrade in a staging environment first.

---

## 11. Verify

### Health check endpoint

```bash
curl -s http://localhost:3000/api/health
```

Expected response:

```json
{"commit":"abc1234","database":"ok","version":"12.4.2"}
```

A `200 OK` with `"database":"ok"` confirms Grafana is running and connected to its backing store.

### Login to the web UI

1. Open `http://localhost:3000` in a browser.
2. Log in with `admin` / `admin` (or the password you set).
3. Navigate to **Connections > Data sources** and confirm Prometheus appears with a green status indicator.

### Verify Prometheus data source via API

```bash
curl -s http://admin:admin@localhost:3000/api/datasources | python3 -m json.tool
```

### Run a test query

1. Go to **Explore** in the Grafana side menu.
2. Select the **Prometheus** data source.
3. Enter `up` in the query editor and click **Run query**.
4. Confirm that results appear showing the status of monitored targets.
