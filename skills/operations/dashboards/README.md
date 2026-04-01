# Grafana Dashboards

> Grafana version 12.x unless noted. Official docs: <https://grafana.com/docs/grafana/latest/dashboards/>

## Create a Dashboard

Step-by-step UI workflow:

1. Open the Grafana web UI and click **Dashboards** in the left-hand navigation menu.
2. Click **New** then **New Dashboard**.
3. On the empty dashboard canvas, click **+ Add visualization**.
4. In the data-source picker dialog, select your Prometheus data source (or any configured source).
5. In the query editor, write or build a query:
   ```promql
   rate(http_requests_total{job="myapp"}[5m])
   ```
6. Click **Run queries** (or press Shift+Enter) to preview the result.
7. In the right-hand panel sidebar, choose a visualization type (e.g., Time series, Stat, Gauge).
8. Configure panel options:
   - **Title** -- give the panel a descriptive name.
   - **Standard options** -- unit, min/max, decimals, display name.
   - **Thresholds** -- color-coded value boundaries.
   - **Value mappings** -- map specific values to text/colors.
   - **Field overrides** -- per-series customization.
9. Click **Back to dashboard** (arrow in the top-left) to return to the canvas.
10. Repeat steps 3-9 to add more panels.
11. Drag panels to rearrange and resize them. Group panels into rows via **Add** > **Row**.
12. Click **Save dashboard** (Ctrl+S / Cmd+S).
13. Enter a dashboard **title**, optional **description**, and select a **folder**.
14. Click **Save**.

---

## Panel Types

| Panel Type | Use Case | Data Requirements |
|---|---|---|
| **Time series** | Metrics over time, trends, rate graphs | Numeric values with timestamps |
| **Stat** | Single aggregate value (current, mean, max) | One or more numeric series |
| **Gauge** | Current value against a min/max range | Single numeric value per series |
| **Bar gauge** | Horizontal/vertical bars showing relative magnitudes | One or more numeric series |
| **Table** | Tabular view of raw or transformed data | Any query result (instant or range) |
| **Heatmap** | Distribution of values over time (latency histograms) | Histogram buckets or raw numeric series |
| **Pie chart** | Proportional breakdown of categories | Labeled numeric values |
| **Bar chart** | Categorical comparison (e.g., per-endpoint counts) | Labeled numeric values |
| **Logs** | Log lines from Loki, Elasticsearch, etc. | Log-type query results |
| **State timeline** | Discrete state transitions over time | String or numeric state values with timestamps |
| **Status history** | Status of multiple entities over time | String or numeric state values with timestamps |
| **Histogram** | Distribution of values as a frequency chart | Numeric series |
| **Geomap** | Geospatial data on a world map | Latitude, longitude, and value fields |
| **Canvas** | Free-form, layered custom layout | Any data; manual element placement |
| **XY chart** | Scatter/bubble plots, non-time x-axis | Two or more numeric fields |
| **Trend** | Sparkline-style compact time series | Numeric values with timestamps |
| **Flame graph** | CPU/memory profiling traces | Profiling data (e.g., from Pyroscope) |
| **Traces** | Distributed traces from Tempo, Jaeger, Zipkin | Trace-type query results |
| **Node graph** | Relationship/dependency topology | Node and edge data frames |
| **Alert list** | Active Grafana alert instances | None (reads from alerting engine) |
| **Annotation list** | Dashboard and global annotations | None (reads from annotation store) |
| **Dashboard list** | Links to other dashboards by tag or search | None (reads from Grafana API) |
| **Text** | Markdown/HTML content blocks | None (static content) |
| **News** | RSS/Atom feed reader | RSS feed URL |

---

## Variables

Variables appear as drop-down selectors at the top of the dashboard. Configure them under **Dashboard Settings** > **Variables**.

### Query Variable

Dynamically fetches values from a data source. For Prometheus, use the `label_values` function.

**Example -- list all `instance` labels for a given job:**

| Setting | Value |
|---|---|
| Name | `instance` |
| Type | Query |
| Data source | Prometheus |
| Query | `label_values(up{job="$job"}, instance)` |
| Refresh | On time range change |
| Sort | Alphabetical (asc) |
| Multi-value | Enable |
| Include All option | Enable |

Other useful Prometheus query-variable functions:

```text
label_values(metric_name, label_name)
label_values(label_name)
metrics(regex)
query_result(up{job="myapp"})
```

### Custom Variable

Manually defined, comma-separated list of values.

| Setting | Value |
|---|---|
| Name | `environment` |
| Type | Custom |
| Values | `production,staging,development` |

Key:value syntax for display name mapping:

```text
Production : production, Staging : staging, Development : development
```

### Datasource Variable

Allows switching the data source for the entire dashboard from the drop-down.

| Setting | Value |
|---|---|
| Name | `datasource` |
| Type | Datasource |
| Data source type | Prometheus |

Use `$datasource` as the data source in every panel query.

### Interval Variable

Pre-defined time intervals for aggregation windows.

| Setting | Value |
|---|---|
| Name | `interval` |
| Type | Interval |
| Values | `1m,5m,15m,1h,6h,1d` |
| Auto option | Enable |
| Step count | 30 |

**Built-in interval variables (always available, no configuration needed):**

| Variable | Meaning |
|---|---|
| `$__interval` | Calculated interval based on the time range and panel width (pixel count). Ensures at most one data point per pixel. |
| `$__interval_ms` | Same as `$__interval` but in milliseconds. |
| `$__rate_interval` | `max(4 * scrape_interval, $__interval)`. Recommended for `rate()` and `increase()` in Prometheus to avoid gaps. |
| `$__range` | Duration of the dashboard time range (e.g., `1h`). |
| `$__range_s` | Same in seconds. |

**Using `$__rate_interval` in a Prometheus query:**

```promql
rate(http_requests_total{job="$job", instance=~"$instance"}[$__rate_interval])
```

---

## Template Syntax

### Basic Substitution

Reference a variable with `$variable` or `${variable}`:

```promql
up{job="$job"}
up{job="${job}"}
```

### Formatting Options

Use `${variable:format}` to control how multi-value selections are rendered:

| Syntax | Output for selection `a`, `b` | Typical Use |
|---|---|---|
| `${variable}` | `a,b` (default) | Prometheus label matchers |
| `${variable:regex}` | `a\|b` | Regex alternation in `=~` matchers |
| `${variable:pipe}` | `a\|b` | Same as regex |
| `${variable:csv}` | `a,b` | CSV lists |
| `${variable:json}` | `["a","b"]` | JSON arrays |
| `${variable:doublequote}` | `"a","b"` | Quoted lists |
| `${variable:singlequote}` | `'a','b'` | Quoted lists |
| `${variable:sqlstring}` | `'a','b'` | SQL IN clauses |
| `${variable:glob}` | `{a,b}` | Glob patterns |
| `${variable:text}` | Display text of selection | Titles, descriptions |
| `${variable:queryparam}` | `var-variable=a&var-variable=b` | URL query parameters |
| `${variable:raw}` | Raw, unescaped value | Advanced use |

**Example -- regex formatting in a Prometheus selector:**

```promql
http_requests_total{instance=~"${instance:regex}"}
```

### Repeat Panels and Rows

Repeat a panel for each value of a multi-value variable:

1. Edit the panel.
2. Open the **Overrides / Repeat options** section (or the panel header menu > **More** > **Repeat**).
3. Set **Repeat by variable** to the desired variable (e.g., `instance`).
4. Choose **Direction**: Horizontal or Vertical.
5. Set **Max per row** (for horizontal repeats).

Repeat an entire row:

1. Hover over the row title and click the gear icon.
2. Set **Repeat for** to the variable.
3. All panels inside the row are duplicated per variable value.

---

## Dashboard JSON Model

Every Grafana dashboard is stored as a JSON document. View it via **Dashboard Settings** > **JSON Model**, or export via **Share** > **Export** > **Save to file**.

### Key Fields

```json
{
  "id": null,
  "uid": "abc123xyz",
  "title": "My Application Dashboard",
  "description": "Monitors HTTP traffic and error rates",
  "tags": ["prometheus", "http"],
  "timezone": "browser",
  "editable": true,
  "graphTooltip": 1,
  "schemaVersion": 40,
  "version": 0,
  "refresh": "30s",
  "time": {
    "from": "now-1h",
    "to": "now"
  },
  "fiscalYearStartMonth": 0,
  "templating": {
    "list": [
      {
        "name": "job",
        "type": "query",
        "datasource": { "type": "prometheus", "uid": "PBFA97CFB590B2093" },
        "query": "label_values(up, job)",
        "refresh": 2,
        "multi": false,
        "includeAll": false
      }
    ]
  },
  "panels": [
    {
      "id": 1,
      "type": "timeseries",
      "title": "Request Rate",
      "gridPos": { "h": 8, "w": 12, "x": 0, "y": 0 },
      "datasource": { "type": "prometheus", "uid": "PBFA97CFB590B2093" },
      "targets": [
        {
          "expr": "rate(http_requests_total{job=\"$job\"}[$__rate_interval])",
          "legendFormat": "{{instance}}"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "reqps",
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "green", "value": null },
              { "color": "red", "value": 1000 }
            ]
          }
        },
        "overrides": []
      },
      "options": {}
    }
  ],
  "annotations": {
    "list": [
      {
        "name": "Deployments",
        "datasource": { "type": "prometheus", "uid": "PBFA97CFB590B2093" },
        "expr": "changes(process_start_time_seconds{job=\"$job\"}[2m]) > 0",
        "enable": true
      }
    ]
  },
  "links": []
}
```

### Field Reference

| Field | Type | Description |
|---|---|---|
| `id` | number/null | Database-assigned numeric ID. Set to `null` when importing. |
| `uid` | string | Unique identifier, 8-40 characters. Stable across exports/imports. |
| `title` | string | Dashboard display name. |
| `tags` | string[] | Labels for filtering and search. |
| `timezone` | string | `"browser"`, `"utc"`, or an IANA timezone. |
| `editable` | boolean | Whether viewers with edit permissions can modify the dashboard. |
| `graphTooltip` | number | `0` = default, `1` = shared crosshair, `2` = shared tooltip. |
| `schemaVersion` | number | JSON schema version. Grafana migrates older versions on import. |
| `version` | number | Incremented on each save. Used for optimistic locking. |
| `refresh` | string | Auto-refresh interval (e.g., `"5s"`, `"1m"`, `""`). |
| `time.from` / `time.to` | string | Default time range (e.g., `"now-6h"` / `"now"`). |
| `templating.list` | array | Variable definitions. |
| `panels` | array | Panel objects with `type`, `targets`, `gridPos`, `fieldConfig`. |
| `annotations.list` | array | Annotation query definitions. |

### Export and Import

**Export from UI:**

Dashboard > **Share** (icon) > **Export** > **Save to file**. Optionally toggle **Export for sharing externally** to replace data-source UIDs with `${DS_PROMETHEUS}` placeholders.

**Import from UI:**

**Dashboards** > **New** > **Import** > paste JSON or upload file > map data sources > **Import**.

**Import via API:**

```bash
curl -s -X POST http://localhost:3000/api/dashboards/db \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $GRAFANA_API_TOKEN" \
  -d '{
    "dashboard": '"$(cat my-dashboard.json)"',
    "overwrite": true,
    "message": "Imported via API"
  }'
```

---

## Provisioning Dashboards from Files

Provisioning loads dashboards from local JSON files at startup and keeps them in sync automatically. This is the recommended approach for GitOps workflows.

> Official docs: <https://grafana.com/docs/grafana/latest/administration/provisioning/#dashboards>

### Directory Structure

```text
/etc/grafana/
  provisioning/
    dashboards/
      default.yaml            # provider config
    datasources/
      default.yaml
  dashboards/                  # JSON files referenced by the provider
    overview.json
    http-traffic.json
    node-exporter/
      cpu.json
      memory.json
      disk.json
```

### Provider Configuration

Create a YAML file in the provisioning dashboards directory. Grafana reads all `*.yaml` files in this folder.

**/etc/grafana/provisioning/dashboards/default.yaml:**

```yaml
apiVersion: 1

providers:
  - name: default
    orgId: 1
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    allowUIUpdates: false
    options:
      path: /etc/grafana/dashboards
      foldersFromFilesStructure: true
```

### Provider Field Reference

| Field | Type | Default | Description |
|---|---|---|---|
| `name` | string | required | Unique provider name. |
| `orgId` | number | `1` | Organization ID. |
| `type` | string | `file` | Provider type. Only `file` is built-in. |
| `disableDeletion` | boolean | `false` | When `true`, dashboards removed from disk are not deleted from Grafana. |
| `updateIntervalSeconds` | number | `10` | How often Grafana scans the path for changes. |
| `allowUIUpdates` | boolean | `false` | When `false`, provisioned dashboards cannot be saved from the UI. |
| `options.path` | string | required | Absolute path to the directory containing JSON dashboard files. |
| `options.foldersFromFilesStructure` | boolean | `false` | When `true`, subdirectory names become Grafana folder names. |

### Auto-Reload Behavior

Grafana watches the provider path directory on an interval defined by `updateIntervalSeconds`. When a JSON file is added, modified, or removed, Grafana automatically creates, updates, or deletes the corresponding dashboard. No restart is required.

Set `updateIntervalSeconds: 10` for near-real-time sync during development. Use `30` or higher in production to reduce disk I/O.

### Docker / Docker Compose Example

Mount dashboard files and the provider config into the Grafana container:

```yaml
# docker-compose.yaml (Grafana service snippet)
services:
  grafana:
    image: grafana/grafana:12.4.0
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning
      - ./grafana/dashboards:/etc/grafana/dashboards
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
```

```text
project/
  grafana/
    provisioning/
      dashboards/
        default.yaml
      datasources/
        prometheus.yaml
    dashboards/
      overview.json
      http-traffic.json
```

### Kubernetes ConfigMap Example

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboard-provider
  namespace: monitoring
data:
  default.yaml: |
    apiVersion: 1
    providers:
      - name: default
        orgId: 1
        type: file
        disableDeletion: false
        updateIntervalSeconds: 30
        allowUIUpdates: false
        options:
          path: /var/lib/grafana/dashboards
          foldersFromFilesStructure: true
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboards
  namespace: monitoring
data:
  overview.json: |
    {
      "uid": "overview",
      "title": "Overview",
      "schemaVersion": 40,
      "panels": [],
      "time": { "from": "now-1h", "to": "now" }
    }
```

Mount both ConfigMaps in the Grafana Deployment:

```yaml
volumes:
  - name: dashboard-provider
    configMap:
      name: grafana-dashboard-provider
  - name: dashboards
    configMap:
      name: grafana-dashboards
volumeMounts:
  - name: dashboard-provider
    mountPath: /etc/grafana/provisioning/dashboards
  - name: dashboards
    mountPath: /var/lib/grafana/dashboards
```

---

## Dashboard as Code

### Terraform -- grafana_dashboard Resource

Use the `grafana` Terraform provider to manage dashboards declaratively.

```hcl
terraform {
  required_providers {
    grafana = {
      source  = "grafana/grafana"
      version = ">= 3.0.0"
    }
  }
}

provider "grafana" {
  url  = "http://localhost:3000"
  auth = var.grafana_api_token
}

resource "grafana_folder" "services" {
  title = "Services"
}

resource "grafana_dashboard" "http_traffic" {
  folder    = grafana_folder.services.id
  overwrite = true

  config_json = file("${path.module}/dashboards/http-traffic.json")
}
```

Apply:

```bash
terraform init
terraform plan
terraform apply
```

To manage the JSON inline (useful for templating with `jsonencode`):

```hcl
resource "grafana_dashboard" "simple" {
  folder    = grafana_folder.services.id
  overwrite = true

  config_json = jsonencode({
    uid           = "simple-dashboard"
    title         = "Simple Dashboard"
    schemaVersion = 40
    panels = [
      {
        id    = 1
        type  = "stat"
        title = "Up"
        gridPos = { h = 4, w = 6, x = 0, y = 0 }
        targets = [
          {
            datasource = { type = "prometheus", uid = "prometheus" }
            expr       = "up{job=\"myapp\"}"
          }
        ]
      }
    ]
    time = { from = "now-1h", to = "now" }
  })
}
```

### Grafana API -- Create / Update Dashboard

**Create or update a dashboard with curl:**

```bash
curl -s -X POST http://localhost:3000/api/dashboards/db \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $GRAFANA_API_TOKEN" \
  -d '{
    "dashboard": {
      "uid": "api-created",
      "title": "API-Created Dashboard",
      "schemaVersion": 40,
      "panels": [
        {
          "id": 1,
          "type": "timeseries",
          "title": "CPU Usage",
          "gridPos": { "h": 8, "w": 12, "x": 0, "y": 0 },
          "datasource": { "type": "prometheus", "uid": "prometheus" },
          "targets": [
            {
              "expr": "100 - (avg by(instance) (rate(node_cpu_seconds_total{mode=\"idle\"}[5m])) * 100)",
              "legendFormat": "{{instance}}"
            }
          ],
          "fieldConfig": {
            "defaults": { "unit": "percent", "min": 0, "max": 100 }
          }
        }
      ],
      "time": { "from": "now-1h", "to": "now" }
    },
    "folderUid": "",
    "overwrite": true,
    "message": "Created via API"
  }'
```

**Generate an API token (Service Account):**

```bash
# Create a service account
curl -s -X POST http://localhost:3000/api/serviceaccounts \
  -H "Content-Type: application/json" \
  -H "Authorization: Basic $(echo -n 'admin:admin' | base64)" \
  -d '{"name": "dashboard-deployer", "role": "Editor"}'

# Create a token for the service account (use the id from the response above)
curl -s -X POST http://localhost:3000/api/serviceaccounts/1/tokens \
  -H "Content-Type: application/json" \
  -H "Authorization: Basic $(echo -n 'admin:admin' | base64)" \
  -d '{"name": "deploy-token"}'
```

**Fetch an existing dashboard:**

```bash
curl -s http://localhost:3000/api/dashboards/uid/api-created \
  -H "Authorization: Bearer $GRAFANA_API_TOKEN" | jq .
```

**Delete a dashboard:**

```bash
curl -s -X DELETE http://localhost:3000/api/dashboards/uid/api-created \
  -H "Authorization: Bearer $GRAFANA_API_TOKEN"
```

**Search dashboards:**

```bash
curl -s "http://localhost:3000/api/search?type=dash-db&query=CPU" \
  -H "Authorization: Bearer $GRAFANA_API_TOKEN" | jq .
```

---

## Version Notes

### Grafana 12.x Changes

- **Scenes-powered dashboards** are the default rendering engine in v12. The legacy Angular-based dashboard runtime is fully removed. Dashboards load faster and support new layout primitives.
- **Dashboard schema version 40+** is used by default. Older schemas are auto-migrated on import.
- **Subfolders** (nested folder hierarchy) introduced in v11.4 are now stable. Provisioning with `foldersFromFilesStructure: true` creates nested folders matching the directory tree.
- **New panel types**: Canvas improvements, XY chart GA, Trend panel.
- **Explore to Dashboard** workflow lets you convert an Explore query directly into a dashboard panel.
- The `POST /api/dashboards/db` endpoint now returns `folderUid` instead of `folderId` in responses (though `folderId` is still accepted in requests for backwards compatibility).
- **Dashboard versioning** stores diffs instead of full snapshots, reducing database storage.
- **Public dashboards** require an explicit feature toggle and have moved to a stable API (`/api/dashboards/public`).

### Grafana 11.x Changes

- **Scenes migration** began in v11.0 (opt-in). Set feature toggle `dashboardScene` to enable the new renderer.
- **Angular plugin deprecation**: Angular-based panels show a warning banner. Migration to React equivalents is required before upgrading to v12.
- **Dashboard schema version 37-39** is typical for v11-era dashboards.
- **Folder UID** (`folderUid`) was introduced alongside the legacy numeric `folderId`. Both are accepted in the API; `folderUid` is preferred.
- **Alerting integration**: Unified alerting is the only option (legacy alerting removed in v11). Alert rules are stored separately from dashboards but can be linked via panel annotations.
- **Transformations** expanded with new join, filter, and calculation types.
- **Variable improvements**: Adhoc filter variable supports more data sources. `$__rate_interval` is recommended over manual `$__interval` for Prometheus `rate()`.

### Upgrade Considerations (v11 to v12)

| Area | Action |
|---|---|
| Angular panels | Replace with React equivalents before upgrading. |
| `folderId` in API calls | Switch to `folderUid`. |
| Custom panel plugins | Rebuild against `@grafana/data` and `@grafana/ui` v12 packages. |
| Provisioning configs | No changes required; existing YAML files are compatible. |
| Dashboard JSON | Imported dashboards auto-migrate `schemaVersion`. No manual edits needed. |
