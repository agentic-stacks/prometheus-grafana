# Diagnose: Grafana

Troubleshooting guide for Grafana with decision-tree diagnostics. **Also check:** `skills/reference/known-issues/` for version-specific problems.

**References:**
- https://grafana.com/docs/grafana/latest/troubleshooting/
- https://grafana.com/docs/grafana/latest/setup-grafana/configure-grafana/

---

## 1. Health Check

**Endpoint:** `GET /api/health`

```bash
curl -s http://localhost:3000/api/health | jq .
```

**Expected healthy response:**

```json
{
  "commit": "<git sha>",
  "database": "ok",
  "version": "<grafana version>"
}
```

**Decision tree:**

```
Is /api/health reachable?
├── NO  → Is the Grafana process running?
│   ├── NO  → Start Grafana:
│   │         systemctl start grafana-server
│   │         OR: docker start grafana
│   │   └── Still won't start? → Check logs (see Section 2)
│   └── YES → Is Grafana bound to the expected port?
│       ├── Check: ss -tlnp | grep 3000
│       └── Check grafana.ini [server] section:
│           http_port = 3000
│           http_addr =        (empty = all interfaces)
├── YES, but "database": "failing"
│   └── See Section 6 (Database Issues)
└── YES, "database": "ok"
    └── Grafana is healthy. Problem is elsewhere.
```

---

## 2. Logs

### Log file locations

| Installation method | Default log path |
|---|---|
| Package (DEB/RPM) | `/var/log/grafana/grafana.log` |
| Binary / tarball | `<install_dir>/data/log` |
| Docker | stdout (use `docker logs`) |
| macOS (Homebrew) | `/usr/local/var/log/grafana/grafana.log` |

### Log configuration

Edit `grafana.ini` (typically `/etc/grafana/grafana.ini`):

```ini
[log]
mode = console file
level = info
# For verbose troubleshooting:
# level = debug

[log.file]
log_rotate = true
max_lines = 1000000
max_size_shift = 28
daily_rotate = true
max_days = 7
```

### Viewing logs

```bash
# Systemd
journalctl -u grafana-server -f

# Docker
docker logs grafana --tail 100 -f

# Docker Compose
docker compose logs grafana --tail 100 -f

# Direct file
tail -f /var/log/grafana/grafana.log
```

### Enable HTTP request logging

```ini
[log]
filters = rendering:debug

# Log all HTTP requests
router_logging = true
```

---

## 3. Data Source Connection Failures

### "Bad Gateway" or "Error reading Prometheus"

```
Error reading Prometheus: Post "http://prometheus:9090/api/v1/query": dial tcp ... connection refused
```

**Decision tree:**

```
"Bad Gateway" or connection error on data source?
├── Is the data source URL correct?
│   ├── Check: Grafana UI → Configuration → Data Sources → select source → URL field
│   ├── Common mistake: using localhost when Grafana is in a container
│   │   └── Fix: use the container/service name (e.g., http://prometheus:9090)
│   └── Verify the URL is reachable FROM the Grafana host:
│       docker exec grafana curl -s http://prometheus:9090/api/v1/status/config
│       OR: curl -s http://localhost:9090/api/v1/status/config
├── Is the proxy setting correct?
│   ├── "Server (default)" = Grafana backend proxies the request
│   └── "Browser" = browser connects directly (CORS must be enabled on target)
│   └── For Prometheus behind Grafana, use "Server (default)"
├── Is Prometheus actually running?
│   ├── curl -s http://prometheus:9090/-/healthy
│   └── Expected: "Prometheus Server is Healthy."
└── Network/firewall?
    ├── docker network inspect <network_name>
    └── Verify both containers are on the same Docker network
```

### "No data" in panels

```
Panel shows "No data" or empty graph
```

**Decision tree:**

```
Panel shows "No data"?
├── Is the time range correct?
│   ├── Check the dashboard time picker (top right)
│   ├── Try "Last 1 hour" or "Last 5 minutes"
│   └── Is the metric actually being scraped in that time window?
├── Test the query directly in Explore:
│   ├── Grafana UI → Explore (compass icon)
│   ├── Select the same data source
│   ├── Enter the exact query from the panel
│   └── Run query — does it return data?
├── Does the metric exist?
│   ├── Query Prometheus directly:
│   │   curl -s http://prometheus:9090/api/v1/label/__name__/values | jq . | grep "metric_name"
│   ├── Or in Grafana Explore, type the metric name — autocomplete shows available metrics
│   └── If metric is missing → check Prometheus targets and scrape config
├── Is the data source working at all?
│   ├── Grafana UI → Configuration → Data Sources → select source → "Save & Test"
│   └── Must show "Data source is working"
└── Variable or template issue?
    ├── Check if dashboard variables resolve correctly
    └── Inspect the query with variables expanded (use Explore tab)
```

---

## 4. Authentication Issues

### Cannot login

**Decision tree:**

```
Cannot login to Grafana?
├── Is this a forgotten admin password?
│   └── Reset it:
│       grafana cli admin reset-admin-password <new-password>
│       # With custom homepath:
│       grafana cli --homepath "/usr/share/grafana" admin reset-admin-password <new-password>
│       # Docker:
│       docker exec grafana grafana cli admin reset-admin-password <new-password>
│       # Then restart:
│       systemctl restart grafana-server
├── Is basic auth enabled?
│   └── Check grafana.ini:
│       [auth.basic]
│       enabled = true
├── LDAP authentication failing?
│   ├── Check grafana.ini:
│   │   [auth.ldap]
│   │   enabled = true
│   │   config_file = /etc/grafana/ldap.toml
│   │   allow_sign_up = true
│   ├── Enable LDAP debug logging:
│   │   [log]
│   │   filters = ldap:debug
│   ├── Test LDAP connectivity:
│   │   ldapsearch -x -H ldap://ldap-server:389 -b "dc=example,dc=com" -D "cn=admin,dc=example,dc=com" -W
│   └── Verify ldap.toml bind_dn and search_filter
├── OAuth authentication failing?
│   ├── Check grafana.ini (example for Google):
│   │   [auth.google]
│   │   enabled = true
│   │   client_id = <value>
│   │   client_secret = <value>
│   │   allowed_domains = example.com
│   ├── Verify root_url is correct (required for OAuth callbacks):
│   │   [server]
│   │   root_url = https://grafana.example.com
│   └── Enable auth debug logging:
│       [log]
│       filters = oauth:debug
└── Check auth-related logs:
    grep -i "auth\|login\|ldap\|oauth" /var/log/grafana/grafana.log
    # Docker:
    docker logs grafana 2>&1 | grep -i "auth\|login\|ldap\|oauth"
```

### Anonymous access

```ini
# grafana.ini — enable anonymous read-only access
[auth.anonymous]
enabled = true
org_name = Main Org.
org_role = Viewer
```

---

## 5. Performance Issues

### Slow dashboard loading

**Decision tree:**

```
Dashboard loads slowly?
├── Reduce the time range
│   ├── Querying months of data is expensive
│   └── Use shorter ranges (1h, 6h, 24h) or aggregated queries
├── Too many panels?
│   ├── Each panel fires a separate query
│   ├── Aim for <20 panels per dashboard
│   └── Split into multiple dashboards with links between them
├── Heavy queries?
│   ├── Use recording rules in Prometheus for frequently-used expensive queries:
│   │   # prometheus.yml rules
│   │   groups:
│   │     - name: dashboard_recording_rules
│   │       interval: 1m
│   │       rules:
│   │         - record: job:http_requests:rate5m
│   │           expr: sum(rate(http_requests_total[5m])) by (job)
│   └── Then query the recorded metric instead of computing it live
├── Mixed data source queries?
│   ├── Panels querying multiple data sources are slower
│   └── Prefer single data source per panel
├── Enable query caching (Grafana Enterprise or via Prometheus):
│   ├── grafana.ini:
│   │   [caching]
│   │   enabled = true
│   └── Or use Prometheus --query.lookback-delta and caching reverse proxy
├── Enable gzip compression:
│   ├── grafana.ini:
│   │   [server]
│   │   enable_gzip = true
│   └── Reduces payload size for large query results
└── Check Grafana backend performance:
    ├── Monitor Grafana's own metrics at /metrics
    ├── Key metrics:
    │   grafana_http_request_duration_seconds_bucket
    │   grafana_datasource_request_duration_seconds_bucket
    │   grafana_api_response_status_total
    └── Profile with:
        curl -s http://localhost:3000/debug/pprof/heap > heap.out
        go tool pprof heap.out
```

### Auto-refresh best practices

```
Avoid low refresh intervals on dashboards with wide time ranges.
├── 1-hour range → 10s refresh is acceptable
├── 24-hour range → 1m refresh minimum
├── 7-day range → disable auto-refresh entirely
└── Configure default in grafana.ini:
    [dashboards]
    min_refresh_interval = 10s
```

---

## 6. Database Issues

### SQLite vs MySQL/PostgreSQL

```ini
# grafana.ini — default SQLite (no external DB needed)
[database]
type = sqlite3
path = grafana.db

# MySQL
[database]
type = mysql
host = 127.0.0.1:3306
name = grafana
user = grafana
password = secret
ssl_mode = true

# PostgreSQL
[database]
type = postgres
host = 127.0.0.1:5432
name = grafana
user = grafana
password = secret
ssl_mode = disable
```

### Database decision tree

```
Database problem?
├── Migration failures
│   ├── Check logs for: "migration failed"
│   │   grep -i "migration" /var/log/grafana/grafana.log
│   ├── Lock contention during migration:
│   │   [database]
│   │   migration_locking = false
│   │   # Only set temporarily to bypass lock issues
│   ├── Ensure only ONE Grafana instance runs during migration
│   └── After migration completes, re-enable locking
├── SQLite corruption
│   ├── Symptoms: "database is locked" or "database disk image is malformed"
│   ├── Stop Grafana first:
│   │   systemctl stop grafana-server
│   ├── Backup the database:
│   │   cp /var/lib/grafana/grafana.db /var/lib/grafana/grafana.db.bak
│   ├── Attempt recovery:
│   │   sqlite3 /var/lib/grafana/grafana.db ".dump" | sqlite3 /var/lib/grafana/grafana-recovered.db
│   │   mv /var/lib/grafana/grafana-recovered.db /var/lib/grafana/grafana.db
│   │   chown grafana:grafana /var/lib/grafana/grafana.db
│   └── Restart Grafana
├── Connection pool issues (MySQL/PostgreSQL)
│   ├── Tune in grafana.ini:
│   │   [database]
│   │   max_open_conn = 100
│   │   max_idle_conn = 100
│   │   conn_max_lifetime = 14400
│   └── Enable SQL query logging for debugging:
│       [database]
│       log_queries = true
└── Encrypt data source passwords (one-time migration)
    grafana cli admin data-migration encrypt-datasource-passwords
```

---

## 7. Plugin Issues

### Plugin management commands

```bash
# List installed plugins
grafana cli plugins ls

# Install a plugin
grafana cli plugins install grafana-piechart-panel

# Install a specific version
grafana cli plugins install grafana-piechart-panel 1.6.4

# Update all plugins
grafana cli plugins update-all

# Update a single plugin
grafana cli plugins update grafana-piechart-panel

# Remove a plugin
grafana cli plugins remove grafana-piechart-panel

# List available remote plugins
grafana cli plugins list-remote
```

### Plugin decision tree

```
Plugin not working?
├── Is the plugin installed?
│   ├── grafana cli plugins ls
│   └── If not listed → install it:
│       grafana cli plugins install <plugin-id>
│       systemctl restart grafana-server
├── Plugin shows as "unsigned"?
│   ├── Grafana blocks unsigned plugins by default
│   ├── To allow specific unsigned plugins, edit grafana.ini:
│   │   [plugins]
│   │   allow_loading_unsigned_plugins = <plugin-id-1>,<plugin-id-2>
│   ├── To set the signature acceptance level:
│   │   [plugins]
│   │   plugin_admin_external_manage_enabled = true
│   └── Restart Grafana after changes
├── Plugin fails to load?
│   ├── Check logs:
│   │   grep -i "plugin" /var/log/grafana/grafana.log
│   ├── Reinstall:
│   │   grafana cli plugins remove <plugin-id>
│   │   grafana cli plugins install <plugin-id>
│   │   systemctl restart grafana-server
│   └── Check plugin compatibility with your Grafana version
├── Custom plugin directory?
│   ├── grafana.ini:
│   │   [paths]
│   │   plugins = /var/lib/grafana/plugins
│   └── Or override with CLI:
│       grafana cli --pluginsDir /custom/path plugins ls
└── Docker plugin installation:
    # Via environment variable
    docker run -e "GF_INSTALL_PLUGINS=grafana-piechart-panel,grafana-clock-panel" grafana/grafana
    # Or mount a volume with pre-installed plugins
    docker run -v /host/plugins:/var/lib/grafana/plugins grafana/grafana
```

---

## Quick Reference: Environment Variable Overrides

Any `grafana.ini` setting can be overridden with environment variables using the pattern:

```
GF_<SECTION>_<KEY>=<value>
```

Examples:

```bash
GF_SERVER_HTTP_PORT=3000
GF_DATABASE_TYPE=postgres
GF_AUTH_ANONYMOUS_ENABLED=true
GF_LOG_LEVEL=debug
GF_SECURITY_ADMIN_PASSWORD=secret
```

This is especially useful for Docker and Kubernetes deployments where editing `grafana.ini` directly is not practical.
