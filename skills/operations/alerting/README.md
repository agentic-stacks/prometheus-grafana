# Alerting

Configure alerting in Prometheus and Grafana to detect problems and notify the right people.

**Related skills:** Deploy Alertmanager (`skills/deploy/alertmanager`), Recording rules for pre-computing alert expressions (`skills/operations/recording-rules`), Troubleshoot alert issues (`skills/diagnose/alertmanager`).

## Prometheus Alerting Rules

### Rule File Syntax

Prometheus alerting rules are defined in YAML files and organized into groups:

```yaml
groups:
  - name: example-alerts
    interval: 30s          # optional: override global evaluation interval
    rules:
      - alert: HighRequestLatency
        expr: job:request_latency_seconds:mean5m{job="myjob"} > 0.5
        for: 10m
        keep_firing_for: 5m
        labels:
          severity: page
        annotations:
          summary: "High request latency on {{ $labels.instance }}"
          description: "Request latency is {{ $value }}s for job {{ $labels.job }}."
```

**Field reference:**

| Field | Required | Description |
|---|---|---|
| `alert` | yes | Alert name |
| `expr` | yes | PromQL expression that triggers when true |
| `for` | no | Duration the condition must hold before firing (pending state) |
| `keep_firing_for` | no | Duration to keep firing after the condition clears (prevents flapping) |
| `labels` | no | Extra labels attached to the alert; overwrites conflicting labels |
| `annotations` | no | Informational labels (summary, description, runbook_url); not used for routing |

### Load Rules in prometheus.yml

Reference rule files in the Prometheus configuration and reload:

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "rules/*.yml"
  - "alerts/*.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - "alertmanager:9093"
```

Reload Prometheus to pick up new or changed rule files:

```bash
# HTTP reload (requires --web.enable-lifecycle flag)
curl -X POST http://localhost:9090/-/reload

# Or send SIGHUP
kill -HUP $(pidof prometheus)
```

Validate rule files before loading:

```bash
promtool check rules rules/alerts.yml
```

### Example Rules

#### Instance Down

```yaml
groups:
  - name: instance-health
    rules:
      - alert: InstanceDown
        expr: up == 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Instance {{ $labels.instance }} down"
          description: "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 5 minutes."
```

#### High CPU Usage

```yaml
groups:
  - name: node-resources
    rules:
      - alert: HighCpuUsage
        expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
          description: "CPU usage is above 80% (current value: {{ $value | printf \"%.1f\" }}%) on {{ $labels.instance }}."
```

#### High Memory Usage

```yaml
      - alert: HighMemoryUsage
        expr: (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 > 85
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage on {{ $labels.instance }}"
          description: "Memory usage is above 85% (current value: {{ $value | printf \"%.1f\" }}%) on {{ $labels.instance }}."
```

#### High Request Latency (histogram_quantile)

```yaml
      - alert: HighRequestLatency
        expr: histogram_quantile(0.99, sum by (le, job) (rate(http_request_duration_seconds_bucket[5m]))) > 1
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "High p99 latency for {{ $labels.job }}"
          description: "99th percentile request latency is above 1s (current value: {{ $value | printf \"%.3f\" }}s) for job {{ $labels.job }}."
```

### Templating in Annotations

Annotation and label values support Go templating with these variables:

| Variable | Description |
|---|---|
| `{{ $labels.<name> }}` | Label value from the alert instance (e.g., `{{ $labels.instance }}`) |
| `{{ $value }}` | Numeric result of the alerting expression |
| `{{ $externalLabels.<name> }}` | External labels set in `global.external_labels` |
| `{{ $externalURL }}` | The Prometheus external URL |

Template functions from Go's `text/template` are available:

```yaml
annotations:
  summary: "Disk usage {{ $value | humanize }}% on {{ $labels.instance }}"
  description: >-
    Instance {{ $labels.instance }} of job {{ $labels.job }}
    has disk usage above threshold.
    Current value: {{ $value | printf "%.2f" }}%.
```

Common template functions: `humanize`, `humanize1024`, `humanizeDuration`, `humanizePercentage`, `humanizeTimestamp`, `printf`, `toUpper`, `toLower`, `title`, `reReplaceAll`, `match`.

---

## Grafana Alerting

### How Unified Alerting Works (v11+)

Grafana unified alerting (default since v11) provides a single system to create, manage, and respond to alerts from one consolidated view. The architecture consists of:

- **Alert rules** -- Queries and conditions evaluated on a schedule against one or more data sources.
- **Alert instances** -- Each unique label set produced by a rule becomes a separate alert instance with its own state (Normal, Pending, Alerting, NoData, Error).
- **Notification policies** -- A routing tree that matches alert labels to determine which contact point receives the notification.
- **Contact points** -- Destinations for alert notifications (Email, Slack, PagerDuty, Webhook, etc.).
- **Silences** -- Suppress notifications for matching alerts during maintenance windows.

Grafana supports two alert rule types:

| Type | Storage | Use Case |
|---|---|---|
| **Grafana-managed** | Grafana database | Recommended. Query any backend data source. Richer feature set. |
| **Data-source-managed** | Prometheus, Mimir, or Loki | Rules stored in the data source. Compatible with Prometheus recording/alerting rules. |

### Create an Alert Rule

#### Via the UI

1. Navigate to **Alerts & IRM** > **Alerting** > **Alert rules**.
2. Click **+ New alert rule**.
3. Enter a rule name.
4. Define the query (select data source, write PromQL/LogQL or use the builder).
5. Define expressions/conditions (reduce, threshold, math).
6. Set the evaluation group and interval (e.g., every 1m).
7. Set the pending period (`for` duration).
8. Configure labels and annotations.
9. Click **Save rule and exit**.

#### Via Provisioning (File)

Place YAML files in Grafana's provisioning directory (`/etc/grafana/provisioning/alerting/`):

```yaml
# /etc/grafana/provisioning/alerting/alert-rules.yml
apiVersion: 1
groups:
  - orgId: 1
    name: my_rule_group
    folder: my_first_folder
    interval: 60s
    rules:
      - uid: my_id_1
        title: my_first_rule
        condition: C
        data:
          - refId: A
            relativeTimeRange:
              from: 600
              to: 0
            datasourceUid: prometheus-uid
            model:
              expr: up == 0
              intervalMs: 1000
              maxDataPoints: 43200
              refId: A
          - refId: B
            relativeTimeRange:
              from: 600
              to: 0
            datasourceUid: '__expr__'
            model:
              type: reduce
              expression: A
              reducer: last
              refId: B
          - refId: C
            relativeTimeRange:
              from: 600
              to: 0
            datasourceUid: '__expr__'
            model:
              type: threshold
              expression: B
              conditions:
                - evaluator:
                    params:
                      - 0
                    type: gt
              refId: C
        noDataState: Alerting
        execErrState: Alerting
        for: 5m
        annotations:
          summary: "Instance is down"
          description: "An instance has been unreachable for 5 minutes."
        labels:
          severity: critical
          team: sre
```

### Notification Policies

Notification policies form a routing tree with a default (root) policy at the top:

```yaml
# /etc/grafana/provisioning/alerting/notification-policies.yml
apiVersion: 1
policies:
  - orgId: 1
    receiver: grafana-default-email
    group_by:
      - grafana_folder
      - alertname
    group_wait: 30s
    group_interval: 5m
    repeat_interval: 4h
    routes:
      - receiver: slack-critical
        matchers:
          - severity = critical
        group_wait: 10s
        continue: false
      - receiver: email-warnings
        matchers:
          - severity = warning
        group_by:
          - alertname
          - team
```

**Routing behavior:**

- Alerts are evaluated top-down against child policies.
- All matchers in a policy must match (AND logic).
- The deepest matching policy handles the alert.
- Set `continue: true` to keep evaluating sibling policies after a match.

**Matcher operators:** `=` (equals), `!=` (not equals), `=~` (regex match), `!~` (regex no match).

**Timing parameters:**

| Parameter | Description | Default |
|---|---|---|
| `group_wait` | How long to buffer alerts before sending the first notification | 30s |
| `group_interval` | How long to wait before sending updates to an existing group | 5m |
| `repeat_interval` | How long to wait before re-sending an unchanged alert group | 4h |

### Contact Points

Configure notification destinations. Grafana supports 25+ integrations.

```yaml
# /etc/grafana/provisioning/alerting/contact-points.yml
apiVersion: 1
contactPoints:
  - orgId: 1
    name: email-team
    receivers:
      - uid: email-team-1
        type: email
        settings:
          addresses: "oncall@example.com;teamlead@example.com"
          singleEmail: true

  - orgId: 1
    name: slack-critical
    receivers:
      - uid: slack-critical-1
        type: slack
        settings:
          url: "https://hooks.slack.com/services/T00/B00/XXXX"
          recipient: "#critical-alerts"
          title: |
            {{ len .Alerts.Firing }} firing, {{ len .Alerts.Resolved }} resolved
          text: |
            {{ range .Alerts }}
            *{{ .Labels.alertname }}* ({{ .Labels.severity }})
            {{ .Annotations.summary }}
            {{ end }}

  - orgId: 1
    name: pagerduty-critical
    receivers:
      - uid: pagerduty-critical-1
        type: pagerduty
        settings:
          integrationKey: "your-pagerduty-integration-key"
          severity: critical
          class: "prometheus"

  - orgId: 1
    name: webhook-generic
    receivers:
      - uid: webhook-generic-1
        type: webhook
        settings:
          url: "https://example.com/api/alerts"
          httpMethod: POST
          maxAlerts: 10
```

### Alert Templates

Grafana notification templates use Go `text/template` syntax. Manage them in **Alerts & IRM** > **Alerting** > **Contact points** > **Notification Templates** tab.

Built-in templates:
- `default.title` -- default notification title
- `default.message` -- default notification message body

Custom template example:

```yaml
# /etc/grafana/provisioning/alerting/templates.yml
apiVersion: 1
templates:
  - orgId: 1
    name: custom_slack_template
    template: |
      {{ define "custom_slack.title" -}}
      [{{ .Status | toUpper }}] {{ .GroupLabels.alertname }}
      {{- end }}

      {{ define "custom_slack.message" -}}
      {{ if gt (len .Alerts.Firing) 0 -}}
      *Firing:*
      {{ range .Alerts.Firing -}}
      - {{ .Labels.alertname }}: {{ .Annotations.summary }}
        Instance: {{ .Labels.instance }}
        Value: {{ .Values }}
      {{ end -}}
      {{ end -}}
      {{ if gt (len .Alerts.Resolved) 0 -}}
      *Resolved:*
      {{ range .Alerts.Resolved -}}
      - {{ .Labels.alertname }}: {{ .Annotations.summary }}
      {{ end -}}
      {{ end -}}
      {{- end }}
```

**Available template variables:**

| Variable | Description |
|---|---|
| `.Alerts` | All alert instances in the notification |
| `.Alerts.Firing` | Alert instances with status "firing" |
| `.Alerts.Resolved` | Alert instances with status "resolved" |
| `.GroupLabels` | Labels used for grouping |
| `.CommonLabels` | Labels common to all alerts |
| `.CommonAnnotations` | Annotations common to all alerts |
| `.Status` | `firing` if any alert is firing, otherwise `resolved` |
| `.ExternalURL` | Grafana external URL |

**Per-alert variables (inside `range .Alerts`):**

| Variable | Description |
|---|---|
| `.Labels` | Alert labels (map) |
| `.Annotations` | Alert annotations (map) |
| `.Status` | Individual alert status |
| `.StartsAt` | Time the alert started firing |
| `.EndsAt` | Time the alert resolved |
| `.Values` | Computed values from expressions |
| `.GeneratorURL` | Link back to the alert rule |
| `.Fingerprint` | Unique identifier for the alert instance |
| `.DashboardURL` | Link to the associated dashboard (if configured) |
| `.PanelURL` | Link to the associated panel (if configured) |

---

## Prometheus vs Grafana Alerting

| Feature | Prometheus Alerting | Grafana Alerting |
|---|---|---|
| **Rule definition** | PromQL in YAML files | UI, provisioning YAML, or Terraform |
| **Data sources** | Prometheus only | Any Grafana data source (Prometheus, Loki, SQL, etc.) |
| **Notification routing** | Alertmanager (separate service) | Built-in notification policies and contact points |
| **Template language** | Go templates in Alertmanager | Go templates in Grafana |
| **Multi-condition rules** | One expression per rule | Multiple queries, reduce, threshold, and math expressions |
| **State management** | Pending / Firing / Resolved | Normal / Pending / Alerting / NoData / Error |
| **HA support** | Alertmanager cluster (gossip) | Grafana HA with database-backed deduplication |
| **GitOps friendly** | Native (plain YAML files) | Provisioning YAML, Terraform provider, or HTTP API |
| **Best for** | Infrastructure-level metrics alerting with existing Alertmanager | Multi-source alerting, teams using Grafana as primary UI |

**When to use Prometheus alerting:** You already run Alertmanager, rules are pure PromQL, and you want plain YAML managed in Git alongside Prometheus config.

**When to use Grafana alerting:** You query multiple data sources, want a UI for rule management, or need centralized alerting across Prometheus, Loki, and other backends.

**Both together:** Use Prometheus alerting for infrastructure-level PromQL rules forwarded to Alertmanager, and Grafana alerting for application-level or multi-source rules with built-in notification management.

---

## Version Notes

### Grafana v12.x

- Unified alerting is the only supported alerting system; legacy alerting has been fully removed.
- Improved alert rule evaluation performance and reduced memory usage.
- Enhanced notification policy UI with visual routing tree editor.
- UTF-8 label matcher support in notification policies.
- Alert rule export available in JSON, YAML, and Terraform HCL formats.

### Grafana v11.x

- Unified alerting became the default (no longer opt-in).
- Legacy alerting deprecated and disabled by default; migration path provided.
- Introduced recording rules for Grafana-managed rules.
- Added `keep_firing_for` support in Grafana-managed alert rules.
- Notification policy `continue` matching support improved.
- Contact point testing from the UI (Grafana Alertmanager only).
- RBAC for alerting resources (alert rules, notification policies, contact points).

---

## References

- [Prometheus Alerting Rules](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/)
- [Prometheus Notification Examples](https://prometheus.io/docs/alerting/latest/notification_examples/)
- [Grafana Alerting](https://grafana.com/docs/grafana/latest/alerting/)
- [Grafana Alerting Rules](https://grafana.com/docs/grafana/latest/alerting/alerting-rules/)
- [Grafana Alerting Provisioning](https://grafana.com/docs/grafana/latest/alerting/set-up/provision-alerting-resources/file-provisioning/)
