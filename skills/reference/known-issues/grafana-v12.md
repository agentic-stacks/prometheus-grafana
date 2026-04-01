# Grafana v12 Known Issues

Breaking changes, known bugs, deprecations, and migration steps for Grafana 12.x releases.

Sources: [Grafana v12 What's New](https://grafana.com/docs/grafana/latest/whatsnew/), [Grafana Breaking Changes](https://grafana.com/docs/grafana/latest/breaking-changes/), [Grafana Changelog](https://github.com/grafana/grafana/blob/main/CHANGELOG.md).

Note: As of Grafana v12.0, breaking changes are published in the What's New pages rather than a dedicated breaking changes page.

---

## v12.0

### Angular framework completely removed
**Symptom:** Angular-based plugins, custom panels, and data source plugins fail to load. Dashboard panels show "Plugin not found" or blank panels. The `angular_support_enabled` configuration option no longer has any effect.
**Cause:** Angular was deprecated in v11.0 (disabled by default) and fully removed in v12.0. The React migration is complete.
**Workaround:** Replace all Angular-based plugins with React alternatives before upgrading:
1. Run `grafana cli plugins ls` and check for Angular dependencies.
2. Use the [detect-angular-dashboards](https://github.com/grafana/detect-angular-dashboards) tool to identify affected dashboards.
3. Install React-based replacements from the Grafana plugin catalog.
**Affected versions:** 12.0 through latest
**Status:** By design; permanent removal

### `editors_can_admin` configuration removed
**Symptom:** Editors who previously could create and manage Grafana Teams lose that ability after upgrade. API calls using this setting are ignored.
**Cause:** Configuration option removed in favor of RBAC-based permission management.
**Workaround:** Before upgrading, grant team management permissions to Editor users via RBAC:
```bash
# Using the Grafana API, assign the teams:admin role to editors
curl -X POST http://localhost:3000/api/access-control/roles \
  -H "Content-Type: application/json" \
  -d '{"name":"team-admin","permissions":[{"action":"teams:create"},{"action":"teams:write","scope":"teams:*"}]}'
```
**Affected versions:** 12.0 through latest
**Status:** By design; permanent removal

### Dashboard v2 schema migration is one-way
**Symptom:** After enabling the `dynamic dashboards` feature flag and saving a dashboard, the dashboard is migrated to schema v2 and cannot be reverted to v1.
**Cause:** The v2 schema introduces structural changes that are not backward-compatible with v1.
**Workaround:** Before migrating production dashboards:
1. Export and backup dashboard JSON via the API or UI.
2. Test on a staging instance first.
3. Do not enable the `dynamic dashboards` feature flag on production until you are confident in the migration.
**Affected versions:** 12.0 through latest
**Status:** Open; Grafana plans to address reversibility in a future version

### `cache_size` metric deprecated and split
**Symptom:** Monitoring queries and alerts referencing the `cache_size` metric return no data or show deprecation warnings in Grafana's own metrics output.
**Cause:** The `cache_size` metric had a duplicate registration bug and has been split into two distinct metrics.
**Workaround:** Update monitoring queries to reference the new replacement metrics. Check the [v12.0 changelog](https://github.com/grafana/grafana/blob/main/CHANGELOG.md) for the exact new metric names.
**Affected versions:** 12.0 (deprecated); planned removal in 13.0
**Status:** Deprecated; removal in 13.0

### Stricter data source UID format enforcement
**Symptom:** Data source creation or update API calls fail with validation errors. Provisioning files that previously worked are rejected.
**Cause:** REST APIs and provisioning now enforce a standardized UID format for data sources (alphanumeric, hyphens, underscores only; max 40 characters).
**Workaround:** Audit existing data source UIDs and update any non-compliant UIDs before upgrading:
```bash
# Check all data source UIDs
curl -s http://localhost:3000/api/datasources | jq '.[].uid'
```
Update provisioning YAML to use compliant UIDs.
**Affected versions:** 12.0 through latest
**Status:** By design; permanent enforcement

### `DataLinksContextMenu` `actions` property removed
**Symptom:** Custom plugins that use the `DataLinksContextMenu` component with the `actions` property throw JavaScript errors. Table visualization context menus may lose custom action items.
**Cause:** The `actions` property was experimental and under a feature flag; it has been removed from the component API.
**Workaround:** Plugin developers must remove references to the `actions` property and implement custom action handling outside the context menu component. Rebuild and redeploy affected plugins.
**Affected versions:** 12.0 through latest
**Status:** By design; permanent removal

---

## v12.1

### Mute Timings renamed to Active Time Intervals
**Symptom:** API calls referencing "mute timings" may behave differently. Documentation and UI now use "Active Time Intervals" terminology.
**Cause:** Renamed to better align with actual usage patterns (defining when notifications are active rather than when they are muted).
**Workaround:** Update automation scripts and documentation to use the new terminology. Check API endpoints for any naming changes.
**Affected versions:** 12.1 through latest
**Status:** By design; permanent rename

---

## v12.2

No breaking changes documented for v12.2. This release focused on new features including SQL expressions (public preview), enhanced table visualization (GA), and canvas pan/zoom improvements.

---

## v12.3 / v12.4

Consult the [What's New](https://grafana.com/docs/grafana/latest/whatsnew/) pages and the [Grafana Changelog](https://github.com/grafana/grafana/blob/main/CHANGELOG.md) for the latest information on these releases.

---

## Upgrade Checklist for v12.0

Before upgrading from v11.x to v12.0:

1. **Audit Angular plugins** -- Replace all Angular-based plugins with React alternatives.
2. **Backup dashboards** -- Export all dashboards as JSON before enabling schema v2.
3. **Review team permissions** -- If using `editors_can_admin`, migrate to RBAC grants.
4. **Validate data source UIDs** -- Ensure all UIDs conform to the new format.
5. **Update monitoring queries** -- Replace `cache_size` metric references.
6. **Review plugin code** -- Remove any `DataLinksContextMenu` `actions` property usage.
