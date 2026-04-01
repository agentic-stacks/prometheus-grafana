# Upgrades

**Before upgrading:** Check `skills/reference/known-issues/` for the target version and `skills/reference/compatibility/` for version interop.

## General Upgrade Strategy

1. **Read release notes** for every version between your current and target version. Pay attention to breaking changes, deprecated features, and required migration steps.
2. **Back up everything** before starting: configuration files, data directories, and databases.
3. **Test in staging** by deploying the new version against a copy of production data and configuration. Run `promtool check config` and validate dashboards and alerts.
4. **Rolling upgrade for HA setups** — upgrade one replica at a time. Confirm the upgraded instance is healthy (serving traffic, scraping targets, evaluating rules) before moving to the next.
5. **Upgrade one component at a time** — upgrade Prometheus first, then Alertmanager, then Grafana. This isolates failures.

## Upgrade Prometheus

### v2.x to v3.x — Key Breaking Changes

**Removed feature flags (now default behavior):**

| Removed Flag | v3 Behavior |
|---|---|
| `promql-at-modifier` | Always enabled |
| `promql-negative-offset` | Always enabled |
| `expand-external-labels` | `${var}` / `$var` in external labels expanded automatically |
| `no-default-scrape-port` | Targets no longer receive automatic port based on scheme |
| `agent` | Use `--agent` CLI flag instead |
| `remote-write-receiver` | Use `--web.enable-remote-write-receiver` flag instead |
| `auto-gomemlimit` | Automatic; disable with `--no-auto-gomemlimit` |
| `auto-gomaxprocs` | Automatic; disable with `--no-auto-gomaxprocs` |

**Configuration changes:**

- `scrape_classic_histograms` renamed to `always_scrape_classic_histograms`
- `http_config.enable_http2` in `remote_write` defaults to `false` (was `true` in v2)
- UTF-8 is now allowed in metric and label names. Set `metric_name_validation_scheme: legacy` globally or per scrape job to preserve v2 behavior.

**PromQL breaking changes:**

- `.` in regular expressions now matches newlines. Replace `foo.*` with `foo[^\n]*` to restore v2 behavior.
- Range selectors are now left-open, right-closed `(start, end]` instead of `[start, end]`. Subqueries like `foo[1m:1m]` may return fewer samples. Extend to `foo[2m:1m]` if needed.
- `holt_winters` renamed to `double_exponential_smoothing` and requires `--enable-feature=promql-experimental-functions`.

**Scrape protocol changes:**

- Missing or invalid `Content-Type` headers cause scrapes to fail. Add `fallback_scrape_protocol` to scrape configs:

```yaml
scrape_configs:
  - job_name: "legacy-target"
    fallback_scrape_protocol: PrometheusText0.0.4
```

**Label normalization:**

- `le` and `quantile` label values are normalized to float representation. Queries using `le="1"` must change to `le="1.0"`.

**Alertmanager compatibility:**

- Prometheus 3 requires Alertmanager >= 0.16.0. Change `api_version: v1` to `api_version: v2` in `alertmanager_config`.

**TSDB format:**

- TSDB format changed in v2.55. Prometheus v3 can only downgrade to v2.55 or newer. Upgrade to v2.55 first if you need a safe rollback path.

**Log format:**

- Logs now use `log/slog` instead of `go-kit/log`. Format changes from `ts=... caller=... level=...` to `time=... level=... source=...`. Update any log-parsing pipelines.

### v2.x to v3.x — Migration Steps

```bash
# 1. Validate current config against v3 rules
promtool check config prometheus.yml

# 2. Check recording and alerting rules
promtool check rules /etc/prometheus/rules/*.yml

# 3. Upgrade to v2.55 first (safe rollback point)
#    Replace binary and restart
cp prometheus /usr/local/bin/prometheus
systemctl restart prometheus

# 4. Verify v2.55 is healthy
promtool check healthy --url=http://localhost:9090

# 5. Back up TSDB data directory
cp -r /var/lib/prometheus /var/lib/prometheus.bak

# 6. Create a TSDB snapshot (non-destructive, no downtime)
curl -X POST http://localhost:9090/api/v1/admin/tsdb/snapshot
# Response: {"status":"success","data":{"name":"20240101T000000Z-abcdef"}}
# Snapshot stored at: /var/lib/prometheus/snapshots/<name>

# 7. Upgrade to v3.x
cp prometheus-3.x /usr/local/bin/prometheus
systemctl restart prometheus

# 8. Verify v3 is healthy and ready
promtool check healthy --url=http://localhost:9090
promtool check ready --url=http://localhost:9090

# 9. Verify targets are being scraped
curl -s http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | {job: .labels.job, health}'
```

### Within v3.x Minor Upgrades

Minor upgrades within v3.x are binary replacements with no configuration changes required:

```bash
# 1. Back up current binary
cp /usr/local/bin/prometheus /usr/local/bin/prometheus.bak

# 2. Validate config with new binary before deploying
./prometheus-new --config.file=prometheus.yml --check-config

# 3. Replace binary and restart
cp prometheus-new /usr/local/bin/prometheus
systemctl restart prometheus

# 4. Verify
promtool check healthy --url=http://localhost:9090
```

For Docker deployments:

```bash
# Pull new image
docker pull prom/prometheus:v3.x.y

# Restart container (data volume persists)
docker stop prometheus
docker rm prometheus
docker run -d --name prometheus \
  -p 9090:9090 \
  -v prometheus-data:/prometheus \
  -v /etc/prometheus:/etc/prometheus \
  prom/prometheus:v3.x.y \
  --config.file=/etc/prometheus/prometheus.yml
```

### Within v2.x Minor Upgrades

Same binary replacement procedure as v3.x minor upgrades. No config changes between minor versions.

### Backup Before Upgrade

**Copy the data directory:**

```bash
# Stop Prometheus for a consistent copy
systemctl stop prometheus
cp -r /var/lib/prometheus /var/lib/prometheus.bak
systemctl start prometheus
```

**TSDB snapshot API (no downtime):**

```bash
# Requires --web.enable-admin-api flag
# Enable it in your service file or CLI args:
#   --web.enable-admin-api

# Create snapshot
curl -X POST http://localhost:9090/api/v1/admin/tsdb/snapshot
# Response: {"status":"success","data":{"name":"20240101T000000Z-abcdef"}}

# Snapshots are stored under the data directory:
ls /var/lib/prometheus/snapshots/
```

**Back up configuration separately:**

```bash
cp /etc/prometheus/prometheus.yml /etc/prometheus/prometheus.yml.bak
cp -r /etc/prometheus/rules/ /etc/prometheus/rules.bak/
```

### Rollback

```bash
# 1. Stop Prometheus
systemctl stop prometheus

# 2. Restore the previous binary
cp /usr/local/bin/prometheus.bak /usr/local/bin/prometheus

# 3. Restore data directory if TSDB format changed
rm -rf /var/lib/prometheus
cp -r /var/lib/prometheus.bak /var/lib/prometheus

# 4. Restore configuration if changed
cp /etc/prometheus/prometheus.yml.bak /etc/prometheus/prometheus.yml

# 5. Start Prometheus
systemctl start prometheus

# 6. Verify
promtool check healthy --url=http://localhost:9090
```

## Upgrade Grafana

### v11.x to v12.x — Key Breaking Changes

**Data source UID format enforcement:**

The `failWrongDSUID` feature flag is enabled by default in v12.0. Data sources with invalid UIDs are rejected. Valid UIDs must contain only `a-zA-Z0-9-_` and be at most 40 characters.

Find affected data sources before upgrading:

```bash
curl -s http://localhost:3000/api/datasources | \
  jq '.[] | select((.uid | test("^[a-zA-Z0-9\\-_]+$") | not) or (.uid | length > 40)) | {id, uid, name, type}'
```

**Annotation table migration:**

A full-table rewrite of the `annotation` table adds a `dashboard_uid` column. This requires 2-3x the current annotation table size in free disk space. Check table size before upgrading:

```sql
-- PostgreSQL
SELECT pg_size_pretty(pg_total_relation_size('annotation'));

-- MySQL
SELECT ROUND(((data_length + index_length) / 1024 / 1024), 2) AS "Size (MB)"
FROM information_schema.TABLES
WHERE table_schema = 'grafana' AND table_name = 'annotation';
```

**Plugin compatibility:**

Stricter version compatibility checks when installing plugins via `grafana cli plugins install`. Plugins declared as incompatible with your Grafana version are rejected.

### Within v12.x Minor Upgrades — Package Upgrade Commands

**Debian / Ubuntu (apt):**

```bash
sudo apt-get update
sudo apt-get install --only-upgrade grafana
# or for Enterprise:
sudo apt-get install --only-upgrade grafana-enterprise
```

**RHEL / Fedora (dnf/yum):**

```bash
sudo dnf update grafana
# or for Enterprise:
sudo dnf update grafana-enterprise
```

**Docker:**

```bash
# Pull the new tag
docker pull grafana/grafana:12.x.y
# or Enterprise:
docker pull grafana/grafana-enterprise:12.x.y

# Restart container (data volume persists)
docker stop grafana
docker rm grafana
docker run -d --name grafana \
  -p 3000:3000 \
  -v grafana-data:/var/lib/grafana \
  -v /etc/grafana:/etc/grafana \
  grafana/grafana:12.x.y
```

**Post-upgrade — update all plugins:**

```bash
grafana cli plugins update-all
sudo systemctl restart grafana-server
```

### Backup

**Configuration files:**

```bash
# Package installations
cp /etc/grafana/grafana.ini /etc/grafana/grafana.ini.bak
cp -r /etc/grafana/provisioning/ /etc/grafana/provisioning.bak/
```

**Plugins directory:**

```bash
cp -r /var/lib/grafana/plugins /var/lib/grafana/plugins.bak
```

**Database — SQLite (default):**

```bash
# Stop Grafana for a consistent copy
sudo systemctl stop grafana-server
cp /var/lib/grafana/grafana.db /var/lib/grafana/grafana.db.bak
sudo systemctl start grafana-server
```

**Database — MySQL:**

```bash
mysqldump -u root -p grafana > grafana_backup.sql
```

**Database — PostgreSQL:**

```bash
pg_dump grafana > grafana_backup.sql
```

### Rollback

```bash
# 1. Install the previous version explicitly
# Debian/Ubuntu:
sudo apt-get install grafana=<previous-version>
# RHEL/Fedora:
sudo dnf downgrade grafana-<previous-version>

# 2. Restore the database
sudo systemctl stop grafana-server
cp /var/lib/grafana/grafana.db.bak /var/lib/grafana/grafana.db
# For MySQL: mysql -u root -p grafana < grafana_backup.sql
# For PostgreSQL: psql grafana < grafana_backup.sql

# 3. Restore configuration
cp /etc/grafana/grafana.ini.bak /etc/grafana/grafana.ini
cp -r /etc/grafana/provisioning.bak/* /etc/grafana/provisioning/

# 4. Restore plugins
rm -rf /var/lib/grafana/plugins
cp -r /var/lib/grafana/plugins.bak /var/lib/grafana/plugins

# 5. Start Grafana
sudo systemctl start grafana-server
```

## Upgrade Alertmanager

### Versioning

Alertmanager follows Prometheus versioning. Alertmanager 0.27+ is the companion release for Prometheus 3.x. Prometheus 3 requires Alertmanager >= 0.16.0.

### Config Format Changes

**API version:** Prometheus 3 uses Alertmanager API v2 exclusively. Update `prometheus.yml`:

```yaml
alerting:
  alertmanagers:
    - api_version: v2
      static_configs:
        - targets:
            - "alertmanager:9093"
```

**UTF-8 matcher syntax (Alertmanager 0.27+):**

Alertmanager 0.27 introduced UTF-8 aware matchers. If your routing or inhibition rules use label names or values with special characters, set `utf8-strict-mode` to true:

```yaml
# alertmanager.yml
route:
  receiver: default
  routes:
    - matchers:
        - alertname="HighMemory"
```

### Upgrade Procedure

```bash
# 1. Back up config
cp /etc/alertmanager/alertmanager.yml /etc/alertmanager/alertmanager.yml.bak

# 2. Back up data directory
cp -r /var/lib/alertmanager /var/lib/alertmanager.bak

# 3. Validate config with new binary
./alertmanager-new --config.file=/etc/alertmanager/alertmanager.yml --check-config

# 4. Replace binary and restart
cp alertmanager-new /usr/local/bin/alertmanager
systemctl restart alertmanager

# 5. Verify
curl -s http://localhost:9093/-/healthy
curl -s http://localhost:9093/-/ready
```

For Docker:

```bash
docker pull prom/alertmanager:v0.28.0
docker stop alertmanager
docker rm alertmanager
docker run -d --name alertmanager \
  -p 9093:9093 \
  -v alertmanager-data:/alertmanager \
  -v /etc/alertmanager:/etc/alertmanager \
  prom/alertmanager:v0.28.0 \
  --config.file=/etc/alertmanager/alertmanager.yml
```

### Rollback

```bash
systemctl stop alertmanager
cp /usr/local/bin/alertmanager.bak /usr/local/bin/alertmanager
cp /etc/alertmanager/alertmanager.yml.bak /etc/alertmanager/alertmanager.yml
rm -rf /var/lib/alertmanager
cp -r /var/lib/alertmanager.bak /var/lib/alertmanager
systemctl start alertmanager
```

## Pre-Upgrade Checklist

- [ ] Read release notes for every version between current and target
- [ ] Check breaking changes and deprecation notices
- [ ] Validate current configuration files: `promtool check config prometheus.yml`
- [ ] Validate recording and alerting rules: `promtool check rules rules/*.yml`
- [ ] Back up Prometheus data directory (`/var/lib/prometheus`) or take a TSDB snapshot via `POST /api/v1/admin/tsdb/snapshot`
- [ ] Back up Prometheus configuration (`prometheus.yml`, `rules/`)
- [ ] Back up Grafana database (`grafana.db`, or dump MySQL/PostgreSQL)
- [ ] Back up Grafana configuration (`grafana.ini`, `provisioning/`)
- [ ] Back up Grafana plugins directory (`/var/lib/grafana/plugins`)
- [ ] Back up Alertmanager configuration (`alertmanager.yml`) and data directory
- [ ] Verify sufficient disk space (especially for Grafana annotation table migration in v12.0)
- [ ] Test the upgrade in a staging environment first
- [ ] For HA setups, plan a rolling upgrade — one replica at a time
- [ ] Confirm Alertmanager version compatibility (>= 0.16.0 for Prometheus 3)
- [ ] After upgrade, verify all targets are being scraped: `curl http://localhost:9090/api/v1/targets`
- [ ] After upgrade, verify alerting pipeline: check Alertmanager is receiving alerts
- [ ] After upgrade, verify Grafana dashboards render correctly and data sources connect
- [ ] After upgrade, update Grafana plugins: `grafana cli plugins update-all`
- [ ] Document the upgrade (version numbers, date, any issues encountered)
