# Grafana v11 Known Issues

Breaking changes, known bugs, deprecations, and migration steps for Grafana 11.x releases.

Sources: [Grafana v11.0 Breaking Changes](https://grafana.com/docs/grafana/latest/breaking-changes/breaking-changes-v11-0/), [Grafana What's New](https://grafana.com/docs/grafana/latest/whatsnew/), [Grafana Changelog](https://github.com/grafana/grafana/blob/main/CHANGELOG.md).

---

## v11.0

### AngularJS support disabled by default
**Symptom:** AngularJS-dependent plugins fail to load. Warning icons appear in the plugin catalog. Dashboard panels that rely on Angular plugins render as blank or show error messages.
**Cause:** AngularJS framework support turned off by default as part of the migration to React. Full removal planned for v12.0.
**Workaround:** For a temporary fix on self-managed instances, re-enable Angular support:
```ini
# grafana.ini
[security]
angular_support_enabled = true
```
Permanent fix: migrate to React-based plugin alternatives. Use the [detect-angular-dashboards](https://github.com/grafana/detect-angular-dashboards) tool to identify affected dashboards.
**Affected versions:** 11.0 through 11.6 (removed entirely in 12.0)
**Status:** Angular fully removed in 12.0; migrate before upgrading

### Legacy alerting completely removed
**Symptom:** Grafana fails to start if legacy alerting configuration is present. Error message indicates legacy alerting is no longer supported.
**Cause:** Legacy alerting system reached end-of-life. The migration path was available through v10.4.x only.
**Workaround:** You must migrate to Grafana Alerting before upgrading to v11.0:
1. Downgrade to v10.4.x if not already on it.
2. Run the built-in legacy-to-unified alerting migration.
3. Verify all alert rules migrated correctly.
4. Then upgrade to v11.0.

There is no migration path from legacy alerting once on v11.0+.
**Affected versions:** 11.0 through latest
**Status:** By design; permanent removal

### Anonymous users counted toward Enterprise license
**Symptom:** License usage increases unexpectedly after upgrade. Billing may change for Grafana Enterprise customers.
**Cause:** Anonymous users are now counted as licensed users in the Enterprise licensing model.
**Workaround:** Disable anonymous access if it inflates licensing costs:
```ini
# grafana.ini
[auth.anonymous]
enabled = false
```
Alternatively, use public dashboards for view-only sharing that does not require user sessions.
**Affected versions:** 11.0 through latest (Enterprise only)
**Status:** By design; permanent licensing change

### Subfolders break alert routing when folder names contain forward slashes
**Symptom:** Alert notifications are sent to the default receiver instead of the configured receiver. Affects alert rules in folders whose names contain `/` characters.
**Cause:** The subfolder feature interprets `/` as a path separator, breaking notification policy folder matchers.
**Workaround:** Before upgrading, update notification policy routes to escape forward slashes:
```
# Before
folder = "MyFolder/sub-folder"
# After
folder = "MyFolder\/sub-folder"
```
Create duplicate routes with escaped matchers before upgrade, then remove old routes after.
**Affected versions:** 11.0 through latest
**Status:** By design; operators must update matchers

### Reporting API deprecated endpoints removed
**Symptom:** API calls to deprecated reporting endpoints return 404. Deprecated field values in the reporting API are rejected.
**Cause:** Old scheduling format endpoints and legacy email/dashboard fields eliminated in API modernization.
**Workaround:** Update API integration code to use the current reporting API endpoints. Consult the [Reporting API documentation](https://grafana.com/docs/grafana/latest/developers/http_api/reporting/).
**Affected versions:** 11.0 through latest (Cloud and Enterprise)
**Status:** By design; permanent removal

### Input data source plugin removed
**Symptom:** Dashboards using the "Direct Input" data source show "Plugin not found" errors.
**Cause:** The direct input data source plugin was removed after four years in alpha. Superseded by TestData.
**Workaround:** Migrate affected dashboard panels to use the TestData data source instead.
**Affected versions:** 11.0 through latest
**Status:** By design; permanent removal

### Hidden queries now filtered before panel display
**Symptom:** Data disappears from panels when queries have "Hide response" toggled on. Custom data sources extending `DataSourceWithBackend` may behave differently.
**Cause:** Query filtering standardization. Hidden queries are now automatically excluded from panel rendering. The button was renamed from "Disable query" to "Hide response / Show response".
**Workaround:** Check all panels for accidentally hidden queries. For custom data sources, implement query migration logic inside the `filterQuery` method.
**Affected versions:** 11.0 through latest
**Status:** By design; permanent behavior change

### Google OAuth HD parameter validation added
**Symptom:** Users fail to log in via Google OAuth. Error messages reference missing or mismatched HD (hosted domain) parameter.
**Cause:** New security validation checks the ID token HD parameter against the configured allowed domains list.
**Workaround:** If using Google OAuth without HD parameter (e.g., personal Gmail accounts), disable validation:
```ini
# grafana.ini
[auth.google]
validate_hd = false
```
**Affected versions:** 11.0 through latest
**Status:** By design; security enhancement

### Repeated panel URLs changed format
**Symptom:** Bookmarked or shared URLs for repeated panels return "Panel not found" errors. URLs like `viewPanel=panel-5` no longer work.
**Cause:** Dashboard architecture migrated to Scenes library, which generates different panel IDs for repeated panels (e.g., `viewPanel=panel-3-clone1`).
**Workaround:** Reopen panels in view mode to generate new URLs. Update any saved bookmarks or embedded links.
**Affected versions:** 11.0 through latest
**Status:** By design; permanent URL format change

### Public dashboard footer shows Grafana logo by default
**Symptom:** The Grafana logo appears on public dashboards even if no branding was previously shown.
**Cause:** Default behavior changed to show Grafana logo footer when no custom logo or text is configured. Footer can no longer be hidden entirely.
**Workaround:** Configure custom footer logo or text in Grafana settings to override the default:
```ini
# grafana.ini
[white_labeling]
login_logo = /path/to/custom-logo.svg
```
**Affected versions:** 11.0 through latest (Cloud Advanced and Enterprise)
**Status:** By design; permanent behavior change

---

## v11.1

### XY chart requires feature toggle
**Symptom:** XY chart panel type is not available or does not render data.
**Cause:** The XY chart visualization requires the `autoMigrateXYChartPanel` feature toggle to be enabled.
**Workaround:** Enable the feature toggle:
```ini
# grafana.ini
[feature_toggles]
enable = autoMigrateXYChartPanel
```
**Affected versions:** 11.1
**Status:** Fixed in later 11.x releases when XY chart became GA

---

## v11.2

### Navigation bookmarks require feature toggle on self-managed
**Symptom:** Navigation bookmark feature is not visible in the Grafana UI on self-managed instances.
**Cause:** Feature requires manual enablement via feature toggle.
**Workaround:** Enable the feature toggle:
```ini
# grafana.ini
[feature_toggles]
enable = pinNavItems
```
**Affected versions:** 11.2 through 11.x
**Status:** Fixed in later releases when feature became GA

### Alert history page requires Loki configuration
**Symptom:** Alert history page shows no data or is unavailable on self-managed and Enterprise instances.
**Cause:** Feature requires both a feature toggle and a Loki data source configured for annotations.
**Workaround:** Enable the feature toggle and configure Loki:
```ini
# grafana.ini
[feature_toggles]
enable = alertingCentralAlertHistory

[unified_alerting.state_history]
backend = loki
loki_remote_url = http://loki:3100
```
**Affected versions:** 11.2 through 11.x (Enterprise/OSS)
**Status:** Fixed in later releases

---

## Deprecations Across v11.x

### React Router v5 deprecated for plugins
**Symptom:** Plugin development build warnings about React Router v5 usage.
**Cause:** Migration to React Router v6 in the Grafana frontend.
**Workaround:** Migrate app plugins to use React Router v6 APIs.
**Affected versions:** 11.0 through 11.6
**Status:** React Router v5 support removed in 12.0

### `grafana/e2e` testing tool deprecated
**Symptom:** Deprecation warnings when using `@grafana/e2e` for plugin testing.
**Cause:** Replaced by `@grafana/plugin-e2e` which uses Playwright instead of Cypress.
**Workaround:** Migrate plugin test suites to `@grafana/plugin-e2e`.
**Affected versions:** 11.0 through 11.6
**Status:** Deprecated; removal planned in future major version

### ArrayVector deprecated
**Symptom:** TypeScript build errors when using `ArrayVector` in plugin code.
**Cause:** `ArrayVector` replaced with plain JavaScript arrays for data frames.
**Workaround:** Replace `new ArrayVector([1, 2, 3])` with plain arrays `[1, 2, 3]`.
**Affected versions:** 11.0 through 11.6
**Status:** Deprecated; removal planned in future major version

### Tempo non-TraceQL search editor removed
**Symptom:** The older search editor tab in the Tempo data source is gone. Existing queries auto-migrate to the TraceQL editor.
**Cause:** TraceQL editor fully supersedes the legacy search editor.
**Workaround:** No action required; queries migrate automatically. Learn TraceQL syntax for new queries.
**Affected versions:** 11.0 through latest
**Status:** By design; permanent removal

### Loki Search tab removed from Tempo data source
**Symptom:** The "Loki Search" tab is no longer available in the Tempo data source query editor.
**Cause:** Low usage relative to TraceQL; functionality available via Trace to Logs feature.
**Workaround:** Use the Trace to Logs feature for log correlation from traces.
**Affected versions:** 11.0 through latest
**Status:** By design; permanent removal

---

## Upgrade Checklist for v11.0

Before upgrading from v10.x to v11.0:

1. **Migrate legacy alerting** -- Must be done on v10.4.x before upgrading. No migration path exists on v11.0+.
2. **Audit Angular plugins** -- Identify and plan replacements using the detect-angular-dashboards tool.
3. **Review folder names** -- Escape forward slashes in folder-based notification policy matchers.
4. **Update reporting API calls** -- Migrate to current API endpoints.
5. **Check Google OAuth config** -- Verify HD parameter settings if using Google authentication.
6. **Update bookmarked panel URLs** -- Repeated panel URLs will change format.
