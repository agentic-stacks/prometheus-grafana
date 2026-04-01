# Diagnose: Alertmanager

Troubleshooting guide for Alertmanager. Each section follows a decision-tree format:
**If symptom -> check cause -> if confirmed -> apply fix.**

**Also check:** `skills/reference/known-issues/` for version-specific problems.

All endpoints, commands, and queries are exact and drawn from official Prometheus/Alertmanager documentation.

---

## 1. Health Check

Alertmanager exposes two lifecycle endpoints on its default port 9093:

| Endpoint | Method | Meaning |
|---|---|---|
| `GET /-/healthy` | GET | Returns **200** when Alertmanager is running. Does not indicate readiness. |
| `GET /-/ready` | GET | Returns **200** when Alertmanager is ready to accept alerts. Returns **503** during startup. |

### Decision Tree

```
If Alertmanager container/process appears running
  -> curl -sf http://localhost:9093/-/healthy
     -> If non-200 or connection refused
        -> Process is not running or not listening on port 9093.
           Check logs: docker logs <container> or journalctl -u alertmanager
           Check port binding: ss -tlnp | grep 9093
     -> If 200
        -> curl -sf http://localhost:9093/-/ready
           -> If 503
              -> Alertmanager is still starting up.
                 Check logs for errors during initialization.
           -> If 200
              -> Alertmanager is healthy and ready.
              -> Verify config is loaded:
                 curl -s http://localhost:9093/api/v2/status | python3 -m json.tool
```

### Reload Configuration

Alertmanager reloads its configuration at runtime via:

```bash
# HTTP POST (no flags required)
curl -X POST http://localhost:9093/-/reload
```

```bash
# Signal-based reload
kill -SIGHUP $(pidof alertmanager)
```

If the new configuration is not well-formed, changes are not applied and an error is logged.

---

## 2. Alerts Not Firing

This problem spans two systems: Prometheus (generates and sends alerts) and Alertmanager (receives and routes alerts).

### Check Prometheus Side First

```
If an expected alert is not appearing in Alertmanager
  -> Step 1: Is the alerting rule loaded?
     curl -s http://localhost:9090/api/v1/rules?type=alert | python3 -m json.tool
     -> If the rule is not in the response
        -> The rule file is not loaded. Check:
           - Rule file is listed in rule_files in prometheus.yml
           - Rule file path and glob pattern match the actual file
           - Validate: promtool check rules /etc/prometheus/rules/*.yml
           - Reload: curl -X POST http://localhost:9090/-/reload

  -> Step 2: What is the evaluation state?
     curl -s http://localhost:9090/api/v1/rules?type=alert | python3 -m json.tool
     Look at the "state" field for the alert rule:
     -> If state is "inactive"
        -> The alert expression is not matching any time series.
           Debug the PromQL expression directly:
           curl -s "http://localhost:9090/api/v1/query?query=<alert_expr>" | python3 -m json.tool
           -> If result is empty
              -> The condition is not met. Check that the metrics exist and thresholds are correct.
           -> If result has data
              -> The expression returns results but the rule may have label matchers
                 that filter them out. Compare the expr in the rule with the query result.
     -> If state is "pending"
        -> The alert condition is true but the "for" duration has not yet elapsed.
           Check the "for" field in the rule definition.
           Wait for the "for" duration to pass. The alert will transition to firing.
           Common issue: "for: 15m" means the condition must be continuously true
           for 15 minutes before the alert fires.
     -> If state is "firing"
        -> Prometheus considers the alert active. Problem is in delivery to Alertmanager.
           Go to Step 3.

  -> Step 3: Is Prometheus connected to Alertmanager?
     curl -s http://localhost:9090/api/v1/alertmanagers | python3 -m json.tool
     -> If activeAlertmanagers is empty
        -> Prometheus has no Alertmanager configured.
           Fix: Add alerting config to prometheus.yml:
             alerting:
               alertmanagers:
                 - static_configs:
                     - targets:
                         - "alertmanager:9093"
           Reload: curl -X POST http://localhost:9090/-/reload
     -> If activeAlertmanagers lists the Alertmanager
        -> Check for send errors:
           curl -s "http://localhost:9090/api/v1/query?query=prometheus_notifications_errors_total" | python3 -m json.tool
           curl -s "http://localhost:9090/api/v1/query?query=prometheus_notifications_dropped_total" | python3 -m json.tool
           -> If errors are increasing
              -> Network issue between Prometheus and Alertmanager.
                 Test connectivity: curl -sf http://alertmanager:9093/-/healthy
                 Check DNS resolution: nslookup alertmanager
```

### Check Alertmanager Side

```
If Prometheus shows the alert is firing but it does not appear in Alertmanager
  -> Check alerts received by Alertmanager:
     curl -s http://localhost:9093/api/v2/alerts | python3 -m json.tool
     # or using amtool:
     amtool --alertmanager.url=http://localhost:9093 alert
     -> If the alert is not listed
        -> Alertmanager never received it. Go back to Prometheus side Step 3.
     -> If the alert is listed
        -> Alertmanager has the alert. The issue is in notification delivery.
           Go to Section 3 (Notifications Not Sending).
```

### Verify via amtool

List all active alerts with extended output:

```bash
amtool --alertmanager.url=http://localhost:9093 -o extended alert
```

Query alerts by label:

```bash
amtool --alertmanager.url=http://localhost:9093 alert query alertname="HighMemoryUsage"
```

Query alerts by regex:

```bash
amtool --alertmanager.url=http://localhost:9093 alert query severity=~"critical|warning"
```

---

## 3. Notifications Not Sending

Alert appears in Alertmanager but the receiver (email, Slack, PagerDuty, etc.) never gets a notification.

### Step 1: Debug Routing

Use `amtool config routes test` to determine which receiver an alert with given labels would match:

```bash
amtool --alertmanager.url=http://localhost:9093 config routes test alertname=HighMemoryUsage severity=critical team=platform
```

This prints the receiver name that the alert would route to. Compare this with the expected receiver.

Show the full routing tree:

```bash
amtool --alertmanager.url=http://localhost:9093 config routes show
```

```
If amtool config routes test shows the wrong receiver
  -> The routing tree is not matching as expected.
     -> Check matchers on route entries:
        - Matchers use the syntax: label="value", label=~"regex", label!="value", label!~"regex"
        - Routes are evaluated top-down; the first matching child route wins.
        - If continue: false (the default), evaluation stops at the first match.
        - If no child matches, the parent route's receiver is used.
     -> Fix: Adjust matchers in alertmanager.yml and reload:
        curl -X POST http://localhost:9093/-/reload
```

### Step 2: Check Receiver Configuration

```
If the correct receiver is matched but notifications are not delivered
  -> Check Alertmanager logs for send errors:
     docker logs <container> 2>&1 | grep -i "error\|fail\|notify"

  -> Common errors by channel:

     Email (smtp):
       -> "dial tcp: lookup smtp.example.com: no such host"
          Fix: Verify smtp_smarthost in global config resolves correctly.
       -> "535 Authentication failed"
          Fix: Check smtp_auth_username and smtp_auth_password in the receiver.
       -> "x509: certificate signed by unknown authority"
          Fix: Set smtp_require_tls: false for testing, or provide correct CA.

     Slack:
       -> "channel_not_found"
          Fix: Verify the channel name in the receiver config (include the # prefix or use channel ID).
       -> "invalid_auth" or "token_revoked"
          Fix: Regenerate the Slack webhook URL or bot token.
       -> HTTP 403/404 from Slack API
          Fix: Check that the api_url (webhook URL) is correct and active.

     PagerDuty:
       -> "Event object is invalid" or HTTP 400
          Fix: Verify routing_key (integration key) is a valid 32-character hex string.
       -> "Rate limited"
          Fix: Increase group_interval and repeat_interval to reduce notification volume.

     Webhook:
       -> "connection refused" or timeout
          Fix: Verify the webhook URL is reachable from the Alertmanager container.
          Test: curl -sf <webhook_url>

     OpsGenie:
       -> "Could not authenticate" or HTTP 422
          Fix: Check the api_key in the receiver config.
```

### Step 3: Check Silences

```
If the alert exists and routing is correct, but no notification is sent
  -> Check if the alert is silenced:
     amtool --alertmanager.url=http://localhost:9093 silence query
     -> If a silence matches the alert labels
        -> The silence is suppressing the notification.
           To view details:
           amtool --alertmanager.url=http://localhost:9093 silence query alertname=HighMemoryUsage
        -> To expire (remove) a specific silence:
           amtool --alertmanager.url=http://localhost:9093 silence expire <silence-id>
        -> To expire all silences matching a query:
           amtool --alertmanager.url=http://localhost:9093 silence expire $(amtool --alertmanager.url=http://localhost:9093 silence query -q alertname=HighMemoryUsage)
        -> To expire all silences:
           amtool --alertmanager.url=http://localhost:9093 silence expire $(amtool --alertmanager.url=http://localhost:9093 silence query -q)
```

### Step 4: Check Inhibition Rules

```
If the alert is not silenced but still not generating notifications
  -> Check inhibition rules in alertmanager.yml.
     Inhibition rules suppress a target alert when a source alert is firing.

     Example rule:
       inhibit_rules:
         - source_matchers:
             - severity="critical"
           target_matchers:
             - severity="warning"
           equal: ['alertname']

     This rule means: if a critical alert is firing, suppress the warning alert
     with the same alertname.

     -> Check if a source alert (matching source_matchers) is currently firing:
        amtool --alertmanager.url=http://localhost:9093 alert query severity=critical
     -> If a source alert is firing and the equal labels match
        -> The target alert is being inhibited. This is working as configured.
           Fix (if unintended): Remove or adjust the inhibit_rules in alertmanager.yml.

     CAUTION: If all label names listed in "equal" are missing from both the
     source and target alerts, the inhibition rule will apply. This is a common
     gotcha that causes unexpected suppression.
```

### Step 5: Check Timing

```
If routing, silences, and inhibitions are all correct
  -> The notification may be delayed by grouping timers:
     - group_wait: Time to wait before sending a notification for a new group (default 30s).
       The first notification for a group is delayed by this amount.
     - group_interval: Time to wait before sending notifications about new alerts
       added to a group for which an initial notification has already been sent (default 5m).
     - repeat_interval: Time to wait before re-sending a notification for an alert
       that has already been sent (default 4h).

  -> If you just created the alert, wait at least group_wait before expecting a notification.
  -> If the alert was already in a group that was notified, wait group_interval.
  -> If the alert was already notified, wait repeat_interval before the next notification.
```

---

## 4. Clustering Issues

Alertmanager supports high availability via a gossip-based cluster. Instances communicate over a mesh using port 9094 (TCP and UDP) by default.

### Symptom: Duplicate Notifications

```
If multiple Alertmanager instances are sending duplicate notifications
  -> Check cluster membership:
     curl -s http://localhost:9093/api/v2/status | python3 -m json.tool
     Look at the "cluster" section for "peers" list.
     -> If peers list is empty or contains only the local instance
        -> Instances are not clustered. Each instance acts independently and will
           send its own notifications, causing duplicates.

        -> Fix: Ensure each instance is started with --cluster.peer pointing to
           at least one other instance:
             alertmanager --config.file=alertmanager.yml \
               --cluster.peer=alertmanager-1:9094 \
               --cluster.peer=alertmanager-2:9094

        -> Check that gossip port 9094 is reachable between instances:
           nc -zv alertmanager-1 9094
           nc -zv alertmanager-2 9094

        -> Check firewall rules allow both TCP and UDP on port 9094.
           The gossip protocol uses both TCP (for reliable state transfer)
           and UDP (for protocol messages).

        -> If using a custom cluster listen address:
           --cluster.listen-address=0.0.0.0:9094
           Ensure this matches what peers are connecting to.

     -> If peers list shows all expected instances
        -> Cluster is formed. Check for network partitions:
           Compare the peers list across all instances. Each instance should
           see all other instances.
           curl -s http://alertmanager-1:9093/api/v2/status | python3 -c "import sys,json; print([p['address'] for p in json.load(sys.stdin)['cluster']['peers']])"
           curl -s http://alertmanager-2:9093/api/v2/status | python3 -c "import sys,json; print([p['address'] for p in json.load(sys.stdin)['cluster']['peers']])"
           -> If the peer lists differ
              -> Network partition. Check connectivity between the split groups.
```

### Symptom: Alerts Only On One Instance

```
If alerts only appear on one Alertmanager instance and not others
  -> Check that Prometheus is configured to send to ALL Alertmanager instances:
     curl -s http://localhost:9090/api/v1/alertmanagers | python3 -m json.tool
     -> activeAlertmanagers should list every Alertmanager instance.
     -> If only one instance is listed
        -> Fix: In prometheus.yml, list all instances:
           alerting:
             alertmanagers:
               - static_configs:
                   - targets:
                       - "alertmanager-0:9093"
                       - "alertmanager-1:9093"
                       - "alertmanager-2:9093"
           Reload: curl -X POST http://localhost:9090/-/reload

  -> Check that all instances use the same configuration:
     # Compare configs across instances:
     diff <(curl -s http://alertmanager-0:9093/api/v2/status | python3 -c "import sys,json; print(json.load(sys.stdin)['config']['original'])") \
          <(curl -s http://alertmanager-1:9093/api/v2/status | python3 -c "import sys,json; print(json.load(sys.stdin)['config']['original'])")
     -> If configs differ
        -> Sync configurations across all instances. Alertmanager does NOT
           replicate configuration automatically. Each instance must load the
           same alertmanager.yml.

  -> Check cluster status endpoint on each instance:
     curl -s http://localhost:9093/api/v2/status | python3 -m json.tool
     -> Verify "cluster" -> "status" is "ready" on all instances.
```

---

## 5. Validate Configuration

### Validate with amtool

The primary tool for validating Alertmanager configuration files:

```bash
amtool check-config alertmanager.yml
```

Expected output on success:

```
Checking 'alertmanager.yml'  SUCCESS
Found:
 - global config
 - route
 - 1 inhibit rules
 - 5 receivers
 - 0 templates
```

Expected output on failure:

```
Checking 'alertmanager.yml'  FAILED
  - no route provided in config
```

```
Checking 'alertmanager.yml'  FAILED
  - yaml: line 12: did not find expected key
```

### Using amtool inside a container

```bash
# If using the official prom/alertmanager image:
docker exec <container> amtool check-config /etc/alertmanager/alertmanager.yml
```

### Decision Tree for Config Validation

```
If Alertmanager fails to start or reload fails
  -> Run amtool check-config:
     amtool check-config alertmanager.yml
     -> If FAILED with YAML error
        -> Fix the YAML syntax at the reported line number.
     -> If FAILED with "no route provided"
        -> The config is missing the required top-level route section.
           Fix: Add a route with at least a default receiver.
     -> If FAILED with "undefined receiver"
        -> A route references a receiver name that is not defined in the receivers section.
           Fix: Add the missing receiver or correct the name.
     -> If SUCCESS
        -> Config syntax is valid. Check Alertmanager logs for runtime errors:
           docker logs <container> 2>&1 | tail -50
           -> If "error loading config" or "invalid configuration"
              -> File permissions or mount issue.
                 Verify: ls -la /etc/alertmanager/alertmanager.yml
           -> If reload returns 200 but behavior does not change
              -> Verify the loaded config via API:
                 curl -s http://localhost:9093/api/v2/status | python3 -c "import sys,json; print(json.load(sys.stdin)['config']['original'])"
              -> Compare with the file on disk.
```

### Test Routing Without Sending

Use `amtool config routes test` to dry-run routing decisions:

```bash
# Test which receiver an alert with these labels would match
amtool --alertmanager.url=http://localhost:9093 config routes test \
  alertname=HighCPU severity=critical team=infra

# Show the full routing tree
amtool --alertmanager.url=http://localhost:9093 config routes show
```

### Test Template Rendering

```bash
amtool template render \
  --template.glob='/etc/alertmanager/templates/*.tmpl' \
  --template.text='{{ template "slack.default.markdown.v1" . }}'
```

---

## 6. Useful Alertmanager Metrics

Alertmanager exposes metrics on `GET /metrics` (port 9093):

| Metric | Purpose |
|---|---|
| `alertmanager_alerts` | Number of active alerts (by state: active, suppressed, unprocessed) |
| `alertmanager_alerts_received_total` | Total alerts received by Alertmanager |
| `alertmanager_alerts_invalid_total` | Total alerts rejected as invalid |
| `alertmanager_notifications_total` | Total notifications sent (by integration) |
| `alertmanager_notifications_failed_total` | Total notification send failures (by integration) |
| `alertmanager_notification_latency_seconds` | Notification send latency histogram |
| `alertmanager_silences` | Number of silences (by state: active, pending, expired) |
| `alertmanager_cluster_members` | Number of connected cluster peers |
| `alertmanager_cluster_messages_received_total` | Gossip messages received |
| `alertmanager_cluster_messages_sent_total` | Gossip messages sent |
| `alertmanager_cluster_health_score` | Cluster health (0 is healthy, higher values indicate issues) |
| `alertmanager_config_last_reload_successful` | Whether last config reload succeeded (1) or failed (0) |

### Quick Diagnostic Queries (PromQL)

Check if last config reload succeeded:

```promql
alertmanager_config_last_reload_successful
```

Notification failure rate by integration:

```promql
rate(alertmanager_notifications_failed_total[5m])
```

Ratio of failed to total notifications:

```promql
rate(alertmanager_notifications_failed_total[5m]) / rate(alertmanager_notifications_total[5m])
```

Number of active silences:

```promql
alertmanager_silences{state="active"}
```

Cluster health (0 = healthy):

```promql
alertmanager_cluster_health_score
```

Number of cluster peers:

```promql
alertmanager_cluster_members
```

---

## References

- [Alertmanager Overview](https://prometheus.io/docs/alerting/latest/alertmanager/)
- [Alertmanager Configuration](https://prometheus.io/docs/alerting/latest/configuration/)
- [Alertmanager Management API](https://prometheus.io/docs/alerting/latest/management_api/)
- [Alertmanager Clients API](https://prometheus.io/docs/alerting/latest/clients/)
- [amtool (GitHub)](https://github.com/prometheus/alertmanager#amtool)
- [Alertmanager API v2 Spec](https://github.com/prometheus/alertmanager/blob/main/api/v2/openapi.yaml)
