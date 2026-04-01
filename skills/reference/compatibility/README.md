# Compatibility Matrix

Version compatibility reference for all stack components. Covers Prometheus, Grafana, Alertmanager, node_exporter, data source plugins, operating systems, and build toolchains.

## Current Versions (as of April 2026)

| Component | Latest Stable | LTS / Maintenance | End-of-Life |
|-----------|--------------|-------------------|-------------|
| Prometheus | 3.10.0 | 3.5.1 (LTS) | 2.x (2.55.1 final) |
| Grafana | 12.4 | 11.6 (previous major) | 10.x |
| Alertmanager | 0.31.1 | -- | -- |
| node_exporter | 1.10.2 | -- | -- |

---

## 1. Prometheus + Grafana

Grafana communicates with Prometheus exclusively through the Prometheus HTTP query API (`/api/v1/query`, `/api/v1/query_range`, etc.). Because the query API has been stable since Prometheus 2.0, **all Prometheus 2.x and 3.x releases work with all Grafana 9.x, 10.x, 11.x, and 12.x releases**. There is no hard version lock between the two projects.

| Prometheus Version | Grafana Version | Status | Notes |
|-------------------|-----------------|--------|-------|
| 3.5 -- 3.10 | 12.x | Fully compatible | Recommended pairing for new deployments |
| 3.5 -- 3.10 | 11.x | Fully compatible | Grafana 11 does not expose Prometheus 3 native histogram UI features |
| 3.0 -- 3.4 | 12.x | Fully compatible | |
| 3.0 -- 3.4 | 11.x | Fully compatible | |
| 2.45 -- 2.55 | 12.x | Fully compatible | |
| 2.45 -- 2.55 | 11.x | Fully compatible | |
| 2.45 -- 2.55 | 10.x | Fully compatible | |
| 2.0 -- 2.44 | 10.x | Fully compatible | Older Grafana versions may lack newer PromQL function support in autocomplete |
| 2.0 -- 2.44 | 9.x | Fully compatible | |
| 1.x | Any | Not supported | Prometheus 1.x used a different query API; Grafana dropped support years ago |

**Key considerations:**

- Grafana also supports any system implementing the Prometheus query API (Thanos, Cortex, Mimir, VictoriaMetrics). The same broad compatibility applies.
- Newer PromQL functions (e.g., `info()` in Prometheus 3.x) will execute correctly on the server side even if Grafana's autocomplete does not yet list them. You can always type them manually.
- The Prometheus data source in Grafana is a core built-in; no plugin installation is required.

---

## 2. Prometheus + Alertmanager

Prometheus and Alertmanager are separate binaries developed in the same GitHub organization. They communicate over the Alertmanager HTTP API (v1 and v2). The API has been stable across all modern releases, so **version alignment is flexible**. However, the projects coordinate releases, and running versions from the same era is recommended for best feature coverage.

| Prometheus Version | Alertmanager Version | Status | Notes |
|-------------------|---------------------|--------|-------|
| 3.8 -- 3.10 | 0.30 -- 0.31 | Recommended | Coordinated release era; full feature parity |
| 3.0 -- 3.7 | 0.27 -- 0.29 | Fully compatible | |
| 2.45 -- 2.55 | 0.26 -- 0.28 | Fully compatible | |
| 2.30 -- 2.44 | 0.23 -- 0.27 | Fully compatible | |
| 2.0 -- 2.29 | 0.15 -- 0.25 | Compatible | Older Alertmanager versions lack UTF-8 label support and newer inhibition features |

**Key considerations:**

- Prometheus sends alerts using the Alertmanager v2 API (stable since Alertmanager 0.16). Any Alertmanager >= 0.16 works with any Prometheus >= 2.4 at the protocol level.
- Alertmanager's v1 API was deprecated in 0.16 and removed in 0.28. If running Alertmanager >= 0.28, ensure Prometheus is >= 2.4 (which uses the v2 API by default).
- Running Alertmanager in high-availability (clustered) mode requires all cluster members to run the same Alertmanager version.

---

## 3. Prometheus + node_exporter

node_exporter exposes a `/metrics` endpoint in the standard Prometheus exposition format. Prometheus scrapes it like any other target. **All node_exporter versions work with all Prometheus 2.x and 3.x versions.** There is no version coupling.

| node_exporter Version | Prometheus Version | Status | Notes |
|----------------------|-------------------|--------|-------|
| 1.10.x | 3.x / 2.x | Fully compatible | Current recommended version |
| 1.9.x | 3.x / 2.x | Fully compatible | |
| 1.7 -- 1.8 | 3.x / 2.x | Fully compatible | |
| 1.5 -- 1.6 | 3.x / 2.x | Fully compatible | Missing newer collectors (e.g., cgroup v2 improvements) |
| 1.0 -- 1.4 | 2.x | Fully compatible | Missing many modern collectors |
| 0.x | 2.x | Deprecated | Pre-1.0 metric names differ; recording rules or relabeling needed to adapt |

**Key considerations:**

- node_exporter follows semantic versioning. Metric name changes between major versions (0.x to 1.x) broke dashboards; within 1.x, metrics are stable.
- node_exporter runs only on Linux and macOS (no Windows support). For Windows, use [windows_exporter](https://github.com/prometheus-community/windows_exporter).
- node_exporter 1.10+ supports the OpenMetrics exposition format, which Prometheus 3.x can negotiate automatically.

---

## 4. Grafana + Data Sources

Grafana ships with these core data sources built in (no plugin installation required). Plugin data sources are installed separately and follow their own versioning.

### Core Built-in Data Sources

| Data Source | Grafana 12.x | Grafana 11.x | Grafana 10.x | Notes |
|-------------|:---:|:---:|:---:|-------|
| Prometheus | Yes | Yes | Yes | Native support, no plugin needed |
| Loki | Yes | Yes | Yes | Log aggregation by Grafana Labs |
| Tempo | Yes | Yes | Yes | Distributed tracing by Grafana Labs |
| Mimir | Yes | Yes | Yes | Via Prometheus data source (Mimir implements the Prometheus API) |
| Alertmanager | Yes | Yes | Yes | For silences and alert management |
| Graphite | Yes | Yes | Yes | |
| InfluxDB | Yes | Yes | Yes | Supports InfluxQL and Flux query languages |
| Elasticsearch | Yes | Yes | Yes | Also works with OpenSearch |
| MySQL | Yes | Yes | Yes | |
| PostgreSQL | Yes | Yes | Yes | |
| Microsoft SQL Server | Yes | Yes | Yes | |
| CloudWatch | Yes | Yes | Yes | AWS monitoring |
| Azure Monitor | Yes | Yes | Yes | |
| Google Cloud Monitoring | Yes | Yes | Yes | |
| Jaeger | Yes | Yes | Yes | Distributed tracing |
| Zipkin | Yes | Yes | Yes | Distributed tracing |
| OpenTSDB | Yes | Yes | Yes | |
| Pyroscope | Yes | Yes | No | Continuous profiling; added in Grafana 11 |

### Notable Plugin Data Sources

These are commonly used but must be installed separately via `grafana-cli plugins install` or provisioning.

| Plugin | Plugin ID | Minimum Grafana | Notes |
|--------|-----------|:-:|-------|
| Infinity | yesoreyeram-infinity-datasource | 9.0 | JSON, CSV, XML, GraphQL generic data source |
| Zabbix | alexanderzobnin-zabbix-app | 10.0 | Zabbix monitoring integration |
| ClickHouse | grafana-clickhouse-datasource | 10.0 | Official Grafana Labs plugin |
| MongoDB | grafana-mongodb-datasource | 10.2 | Enterprise only |
| Datadog | grafana-datadog-datasource | 10.0 | Enterprise only |
| Splunk | grafana-splunk-datasource | 10.0 | Enterprise only |

---

## 5. Operating System Support

Pre-compiled binaries are published for the platforms shown below. Building from source may work on additional platforms but is not officially supported.

| Component | Linux amd64 | Linux arm64 | macOS amd64 | macOS arm64 | Windows amd64 | Notes |
|-----------|:---:|:---:|:---:|:---:|:---:|-------|
| Prometheus | Yes | Yes | Yes | Yes | Yes | Official binaries for all listed platforms |
| Alertmanager | Yes | Yes | Yes | Yes | Yes | Same build matrix as Prometheus |
| node_exporter | Yes | Yes | Yes | Yes | No | Linux and macOS only; use windows_exporter for Windows |
| Grafana | Yes | Yes | Yes | Yes | Yes | Deb, RPM, tarball, Homebrew, MSI packages available |
| Pushgateway | Yes | Yes | Yes | Yes | Yes | |

### Docker Image Availability

| Component | Image | Variants |
|-----------|-------|----------|
| Prometheus | `prom/prometheus` | Default (busybox), `-distroless` (3.10+) |
| Alertmanager | `prom/alertmanager` | Default (busybox) |
| node_exporter | `prom/node-exporter` | Default (busybox) |
| Grafana | `grafana/grafana` | Default (Alpine), `grafana/grafana-oss` |
| Pushgateway | `prom/pushgateway` | Default (busybox) |

All Docker images are published to both Docker Hub and `quay.io`. Images are built for `linux/amd64` and `linux/arm64` architectures.

---

## 6. Go Version Requirements

These are the Go versions required to build each component from source. The version is specified in each project's `go.mod` file and may change with each release.

| Component | Current Go Requirement | Additional Build Dependencies | Source |
|-----------|:---:|---|---|
| Prometheus 3.10 | Go 1.25.0 | Node.js (see `.nvmrc`), npm >= 10 (for web UI) | `go.mod` |
| Alertmanager 0.31 | Go 1.25.0 | None | `go.mod` |
| node_exporter 1.10 | Go 1.25.0 | None | `go.mod` |
| Grafana 12.x | Go 1.25.8 | Node.js 22+, Yarn 4+ (for frontend) | `go.mod` |

**Key considerations:**

- All Prometheus ecosystem components (Prometheus, Alertmanager, node_exporter, Pushgateway, exporters) share a common build toolchain and tend to require the same Go version within a given release window.
- Grafana requires a newer Go patch version and has significantly more build dependencies due to its frontend.
- If building from source, use the exact Go version from `go.mod`, not just "or greater." The `toolchain` directive in `go.mod` may auto-download the correct version if using Go 1.21+.
- Pre-compiled binaries and Docker images eliminate the need to install Go entirely. Building from source is only necessary for custom patches or unsupported platforms.

---

## Sources

All information is derived from:

- [Prometheus GitHub Releases](https://github.com/prometheus/prometheus/releases)
- [Prometheus Download Page](https://prometheus.io/download/)
- [Alertmanager GitHub Releases](https://github.com/prometheus/alertmanager/releases)
- [node_exporter GitHub Releases](https://github.com/prometheus/node_exporter/releases)
- [Grafana Installation Docs](https://grafana.com/docs/grafana/latest/setup-grafana/installation/)
- [Grafana Data Sources Docs](https://grafana.com/docs/grafana/latest/datasources/)
- [Grafana What's New](https://grafana.com/docs/grafana/latest/whatsnew/)
- `go.mod` files from each project's main branch (fetched April 2026)
