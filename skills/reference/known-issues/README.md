# Known Issues Index

Version-specific breaking changes, known bugs, deprecations, and required migration steps for all stack components.

| Component | Version | File | Coverage |
|-----------|---------|------|----------|
| Prometheus | 3.x | [prometheus-v3.md](prometheus-v3.md) | v3.0.0 through v3.11.0 |
| Prometheus | 2.x | [prometheus-v2.md](prometheus-v2.md) | v2.53.0 through v2.55.1 (final v2 line) |
| Grafana | 12.x | [grafana-v12.md](grafana-v12.md) | v12.0 through v12.4 |
| Grafana | 11.x | [grafana-v11.md](grafana-v11.md) | v11.0 through v11.6 |

## Entry Format

Each issue follows this structure:

```
### [Short Description]
**Symptom:** What the operator sees
**Cause:** Why it happens
**Workaround:** Exact steps to fix it
**Affected versions:** x.y.z through x.y.w
**Status:** Open / Fixed in x.y.w
```

## Sources

All entries are derived from official release notes and documentation:

- [Prometheus GitHub Releases](https://github.com/prometheus/prometheus/releases)
- [Prometheus Blog](https://prometheus.io/blog/)
- [Prometheus v3 Migration Guide](https://prometheus.io/docs/prometheus/latest/migration/)
- [Grafana What's New](https://grafana.com/docs/grafana/latest/whatsnew/)
- [Grafana Breaking Changes](https://grafana.com/docs/grafana/latest/breaking-changes/)
