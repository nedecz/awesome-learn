# Alerting

## Table of Contents

1. [Alerting Philosophy](#alerting-philosophy)
   - [Alert on Symptoms Not Causes](#alert-on-symptoms-not-causes)
   - [Alert Fatigue](#alert-fatigue)
   - [The Pages at 3am Test](#the-pages-at-3am-test)
2. [Prometheus Alerting Rules](#prometheus-alerting-rules)
3. [Alert Severity Levels](#alert-severity-levels)
4. [Alertmanager](#alertmanager)
5. [Alert Routing Tree](#alert-routing-tree)
6. [Alertmanager Inhibition](#alertmanager-inhibition)
7. [Alertmanager Silences](#alertmanager-silences)
8. [PagerDuty Integration](#pagerduty-integration)
9. [OpsGenie Integration](#opsgenie-integration)
10. [Slack Notifications](#slack-notifications)
11. [Runbooks](#runbooks)
12. [Dead Man's Switch / Watchdog Alerts](#dead-mans-switch--watchdog-alerts)
13. [Multi-Window Multi-Burn-Rate Alerting for SLOs](#multi-window-multi-burn-rate-alerting-for-slos)
14. [Next Steps](#next-steps)
15. [Version History](#version-history)

---

## Alerting Philosophy

### Alert on Symptoms Not Causes

The most important principle of effective alerting is: **alert on the impact to users, not on the internal state of your systems**. Cause-based alerts produce noise; symptom-based alerts produce signal.

```
Symptoms vs. Causes
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  WRONG: Alert when CPU > 80%              ← cause-based
  RIGHT: Alert when error rate > 1%        ← symptom-based

  "The user doesn't care about your CPU.
   The user cares whether their request succeeded."
```

| Scenario | Cause Alert (❌ avoid) | Symptom Alert (✅ prefer) |
|----------|----------------------|--------------------------|
| Database slow | `mysql_query_duration_p99 > 500ms` | `http_request_duration_p99 > 2s` on the endpoints that use MySQL |
| Memory pressure | `node_memory_available_bytes < 20%` | `oom_killer_actions_total > 0` or service error rate spike |
| Disk filling | `node_filesystem_free_bytes < 10%` | `service_write_errors_total > 0` on services writing to that disk |
| Pod restarting | `kube_pod_container_restarts_total > 5` | `http_requests_total{status=~"5.."}` rate increase on affected service |
| Third-party API slow | `upstream_connect_duration_p99 > 1s` | `checkout_completion_rate < 95%` over 10 minutes |

**The hierarchy of actionable alerts:**

```
Alerting Hierarchy (most actionable → least actionable)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Level 1 — USER IMPACT (always alert)
    "Users cannot complete checkout" (error rate > 5%)
    "Payment page is not responding" (availability < 99%)
    "Login is taking > 10 seconds" (latency SLO breach)

  Level 2 — IMMINENT IMPACT (alert during business hours or page on-call)
    "Error budget 50% consumed in 1 hour" (burn rate alert)
    "Disk will be full in 4 hours" (predictive alert)
    "Certificate expires in 7 days"

  Level 3 — INFORMATIONAL (ticket, no page)
    "Deployment completed" 
    "Memory usage trending up over 7 days"
    "Unused feature flag detected"
```

### Alert Fatigue

**Alert fatigue** occurs when engineers receive so many alerts that they become desensitized, start ignoring alerts, or disable them — including the ones that matter.

**Symptoms of alert fatigue:**
- Engineers silence alerts without investigating
- On-call engineers dread their rotation
- Postmortems reveal "the alert fired but no one looked at it"
- More than 10–20% of fired alerts are immediately closed as "not actionable"

**Consequences:**
1. Real incidents are missed or delayed
2. On-call burnout and engineer attrition
3. Loss of trust in the monitoring system
4. Engineers create shadow notification channels to work around alerts

**How to avoid alert fatigue:**

| Practice | Description |
|----------|-------------|
| Regular alert review | Monthly review: delete any alert that fired but required no action in 30 days |
| Define "actionable" | Every alert must have a specific runbook action. If not, make it a metric/dashboard instead |
| Use severity levels | Not every issue is a page at 3am. Route informational alerts to tickets |
| Set appropriate `for` duration | `for: 5m` prevents flapping; avoid `for: 0s` |
| Inhibit cascading alerts | A single node down should not produce 50 pod alerts |
| Group related alerts | Alertmanager `group_by` reduces notification volume |
| Track alert-to-action rate | Measure what % of pages result in actual remediation actions |

### The "Pages at 3am" Test

Before adding any alert that could wake someone up, apply this test:

```
The 3am Test
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Questions to ask before adding a paging alert:

  1. If this fires at 3am on a Sunday, is it URGENT enough to wake
     an engineer immediately?

  2. Is there a specific action the on-call engineer MUST take
     right now (not "look into it")?

  3. Will the situation get MEANINGFULLY WORSE if no one responds
     for 8 hours?

  4. Does the alert represent USER IMPACT that cannot be tolerated
     overnight?

  If the answer to any of these is NO → route to warning or ticket,
  not to a page.

  RULE: "If it's not worth waking someone up, it's not worth an alert."
        — Google SRE Book
```

---

## Prometheus Alerting Rules

Alerting rules are defined in YAML files and loaded by Prometheus. When an expression evaluates to true for a specified `for` duration, an alert fires and is sent to Alertmanager.

### Rule File Fields Explained

| Field | Required | Description |
|-------|----------|-------------|
| `alert` | Yes | Name of the alert. Convention: PascalCase, descriptive |
| `expr` | Yes | PromQL expression that evaluates to a boolean (non-zero = firing) |
| `for` | No | How long the condition must be true before firing. Prevents flapping |
| `labels` | No | Additional labels attached to the alert (e.g., severity, team) |
| `annotations.summary` | No | Short human-readable description. Used in notifications |
| `annotations.description` | No | Detailed explanation with template variables |
| `annotations.runbook_url` | No | Link to the runbook for this alert |
| `annotations.dashboard_url` | No | Link to the Grafana dashboard |

### Complete `alerting.rules.yaml`

```yaml
# alerting.rules.yaml
groups:
  - name: application.rules
    interval: 30s      # How often to evaluate rules in this group
    rules:

      # ─── HIGH ERROR RATE ──────────────────────────────────────────────────
      - alert: HighErrorRate
        expr: |
          (
            sum(rate(http_requests_total{status=~"5.."}[5m])) by (job, service)
            /
            sum(rate(http_requests_total[5m])) by (job, service)
          ) > 0.01
        for: 5m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "High HTTP error rate on {{ $labels.service }}"
          description: >
            Service {{ $labels.service }} is returning
            {{ printf "%.2f" (mul $value 100) }}% error responses over the last 5 minutes.
            Threshold is 1%. This is affecting real users.
          runbook_url: "https://wiki.example.com/runbooks/high-error-rate"
          dashboard_url: "https://grafana.example.com/d/abc123/service-health?var-service={{ $labels.service }}"

      # ─── HIGH LATENCY ─────────────────────────────────────────────────────
      - alert: HighLatency
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, job, service)
          ) > 2.0
        for: 10m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "P99 latency too high on {{ $labels.service }}"
          description: >
            Service {{ $labels.service }} P99 latency is
            {{ printf "%.2f" $value }}s (threshold: 2s) over the last 10 minutes.
            Users are experiencing slow responses.
          runbook_url: "https://wiki.example.com/runbooks/high-latency"

      # ─── INSTANCE DOWN ────────────────────────────────────────────────────
      - alert: InstanceDown
        expr: up == 0
        for: 2m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "Instance {{ $labels.instance }} is down"
          description: >
            Prometheus target {{ $labels.instance }} of job {{ $labels.job }}
            has been unreachable for more than 2 minutes.
            Check if the service is running and the network is reachable.
          runbook_url: "https://wiki.example.com/runbooks/instance-down"

      # ─── DISK SPACE RUNNING LOW ───────────────────────────────────────────
      - alert: DiskSpaceRunningLow
        expr: |
          (
            node_filesystem_avail_bytes{mountpoint="/", fstype!="tmpfs"}
            / node_filesystem_size_bytes{mountpoint="/", fstype!="tmpfs"}
          ) < 0.15
        for: 5m
        labels:
          severity: warning
          team: infrastructure
        annotations:
          summary: "Disk space low on {{ $labels.instance }}"
          description: >
            Filesystem {{ $labels.mountpoint }} on {{ $labels.instance }} has
            {{ printf "%.1f" (mul $value 100) }}% free space remaining (threshold: 15%).
            Estimated time to full: check the disk usage trend dashboard.
          runbook_url: "https://wiki.example.com/runbooks/disk-space"

      # ─── CRITICAL: disk about to be full ─────────────────────────────────
      - alert: DiskSpaceCritical
        expr: |
          (
            node_filesystem_avail_bytes{mountpoint="/", fstype!="tmpfs"}
            / node_filesystem_size_bytes{mountpoint="/", fstype!="tmpfs"}
          ) < 0.05
        for: 5m
        labels:
          severity: critical
          team: infrastructure
        annotations:
          summary: "Disk space critically low on {{ $labels.instance }}"
          description: >
            Filesystem {{ $labels.mountpoint }} on {{ $labels.instance }} has only
            {{ printf "%.1f" (mul $value 100) }}% free space (threshold: 5%).
            Immediate action required to prevent service outage.
          runbook_url: "https://wiki.example.com/runbooks/disk-space#critical"

      # ─── HIGH MEMORY USAGE ────────────────────────────────────────────────
      - alert: HighMemoryUsage
        expr: |
          (
            1 - (
              node_memory_MemAvailable_bytes
              / node_memory_MemTotal_bytes
            )
          ) > 0.90
        for: 10m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "High memory usage on {{ $labels.instance }}"
          description: >
            Node {{ $labels.instance }} is using
            {{ printf "%.1f" (mul $value 100) }}% of available memory (threshold: 90%)
            for the past 10 minutes. OOM kills may occur soon.
          runbook_url: "https://wiki.example.com/runbooks/high-memory"

      # ─── POD CRASH LOOPING ────────────────────────────────────────────────
      - alert: PodCrashLooping
        expr: |
          increase(kube_pod_container_status_restarts_total[15m]) > 3
        for: 5m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "Pod {{ $labels.pod }} is crash looping"
          description: >
            Pod {{ $labels.namespace }}/{{ $labels.pod }}
            container {{ $labels.container }} has restarted
            {{ printf "%.0f" $value }} times in the last 15 minutes.
            Check pod logs: kubectl logs -n {{ $labels.namespace }} {{ $labels.pod }} --previous
          runbook_url: "https://wiki.example.com/runbooks/pod-crash-looping"

      # ─── SERVICE UNAVAILABLE ──────────────────────────────────────────────
      - alert: ServiceUnavailable
        expr: |
          (
            sum(kube_deployment_status_replicas_available) by (namespace, deployment)
            /
            sum(kube_deployment_spec_replicas) by (namespace, deployment)
          ) < 0.5
        for: 2m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "Service {{ $labels.deployment }} has insufficient replicas"
          description: >
            Deployment {{ $labels.namespace }}/{{ $labels.deployment }} has fewer than
            50% of desired replicas available. Service may be degraded or unavailable.
            Available: {{ $value | humanize }} of desired.
          runbook_url: "https://wiki.example.com/runbooks/service-unavailable"

      # ─── SLO ERROR BUDGET BURNING FAST ───────────────────────────────────
      - alert: SLOErrorBudgetBurning
        expr: |
          (
            sum(rate(http_requests_total{status=~"5.."}[1h])) by (service)
            /
            sum(rate(http_requests_total[1h])) by (service)
          ) > (1 - 0.999) * 14.4
        for: 5m
        labels:
          severity: critical
          team: platform
          slo: "availability-99.9"
        annotations:
          summary: "Error budget burning too fast for {{ $labels.service }}"
          description: >
            Service {{ $labels.service }} is burning through its 30-day error budget
            at 14.4x the sustainable rate. At this pace, the budget will be exhausted
            in ~2 hours. Current error rate: {{ printf "%.4f" (mul $value 100) }}%.
          runbook_url: "https://wiki.example.com/runbooks/error-budget-burn"
```

**Loading rules in Prometheus:**
```yaml
# prometheus.yml
global:
  evaluation_interval: 15s

rule_files:
  - /etc/prometheus/rules/*.yaml
  - /etc/prometheus/rules/alerting.rules.yaml

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093
      timeout: 10s
```

---

## Alert Severity Levels

Severity levels help route alerts to the right channel and set appropriate response expectations.

| Severity | Color | Definition | Response Time SLA | Example |
|----------|-------|------------|-------------------|---------|
| `critical` | 🔴 Red | Production is down or severely degraded; users actively impacted | Immediate; page on-call 24/7 | Error rate > 5%, production outage, data loss |
| `warning` | 🟡 Yellow | Service is degraded but not down; impact is limited or imminent | Respond within business hours or next business day | Disk at 85%, error rate at 0.5%, memory > 80% |
| `info` | 🔵 Blue | Notable event; no immediate action required | Create ticket; address in next sprint | Certificate expires in 30 days, deployment completed |
| `none` | ⬜ Gray | Used for watchdog/heartbeat alerts; should always be firing | N/A — absence of this alert is the alert | Dead man's switch |

```
Severity Routing Decision Tree
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Is production impacted RIGHT NOW?
  ├── YES → critical → Page on-call immediately
  └── NO  → Will it become critical without intervention?
            ├── Within hours → warning → Slack + ticket, no page
            └── Within days  → info    → Ticket only, no Slack

  At any severity: Does this require human action to resolve?
  └── NO → Do not alert. Add to dashboard instead.
```

---

## Alertmanager

Alertmanager receives alerts from Prometheus, groups them, deduplicates them, applies routing rules, and dispatches notifications to the correct receivers.

### Architecture

```
Alertmanager Architecture
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
  │ Prometheus 1 │    │ Prometheus 2 │    │ Prometheus 3 │
  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘
         │                   │                   │
         └───────────────────┼───────────────────┘
                             │ POST /api/v2/alerts
                             ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │                     Alertmanager                                  │
  │                                                                   │
  │  ┌────────────────┐    ┌─────────────────┐   ┌────────────────┐  │
  │  │  API Handler   │───▶│  Dispatcher     │   │  Inhibitor     │  │
  │  │  Dedup/Group   │    │  Route matching │   │  Silencer      │  │
  │  └────────────────┘    └────────┬────────┘   └────────────────┘  │
  │                                 │                                  │
  │                    ┌────────────┼────────────┐                    │
  │                    ▼            ▼            ▼                    │
  │             ┌─────────┐  ┌──────────┐  ┌──────────┐             │
  │             │ Route 1  │  │ Route 2  │  │ Default  │             │
  │             │(critical)│  │(warnings)│  │(catch-all)             │
  │             └────┬─────┘  └────┬─────┘  └────┬─────┘             │
  │                  │             │              │                    │
  └──────────────────┼─────────────┼──────────────┼────────────────── ┘
                     │             │              │
              ┌──────▼─────┐ ┌────▼─────┐ ┌─────▼────────┐
              │  PagerDuty │ │  Slack   │ │    Email     │
              └────────────┘ └──────────┘ └──────────────┘
```

### Complete `alertmanager.yml`

```yaml
# alertmanager.yml
global:
  # How long to wait before re-sending a resolved notification
  resolve_timeout: 5m

  # SMTP settings for email notifications
  smtp_smarthost: "smtp.example.com:587"
  smtp_from: "alertmanager@example.com"
  smtp_auth_username: "alertmanager@example.com"
  smtp_auth_password: "${SMTP_PASSWORD}"
  smtp_require_tls: true

  # Slack webhook (global default, can be overridden per receiver)
  slack_api_url: "${SLACK_WEBHOOK_URL}"

  # PagerDuty API URL
  pagerduty_url: "https://events.pagerduty.com/v2/enqueue"

# Templates for notification formatting
templates:
  - "/etc/alertmanager/templates/*.tmpl"

route:
  # The root route. All alerts start here.
  receiver: "default-receiver"

  # Labels to group alerts by. Alerts with same labels = one notification.
  group_by: ["alertname", "service", "namespace"]

  # Wait this long for more alerts to arrive before sending the first notification
  group_wait: 30s

  # Wait this long before sending a new notification for ongoing group
  group_interval: 5m

  # How long before re-notifying about an ongoing alert
  repeat_interval: 4h

  routes:
    # Critical alerts → PagerDuty (immediate page)
    - match:
        severity: critical
      receiver: pagerduty-critical
      group_wait: 0s          # Don't wait — page immediately
      repeat_interval: 1h
      routes:
        # Override: payments team gets their own PagerDuty service
        - match:
            team: payments
          receiver: pagerduty-payments
        # Override: infrastructure team gets their own
        - match:
            team: infrastructure
          receiver: pagerduty-infrastructure

    # Warning alerts → Slack (no page)
    - match:
        severity: warning
      receiver: slack-warnings
      group_wait: 1m
      group_interval: 10m
      repeat_interval: 12h

    # Info alerts → Slack info channel (no page, low priority)
    - match:
        severity: info
      receiver: slack-info
      group_wait: 5m
      repeat_interval: 24h

    # Watchdog (dead man's switch) — must ALWAYS be firing
    - match:
        alertname: Watchdog
      receiver: deadmanssnitch
      group_wait: 0s
      group_interval: 1m
      repeat_interval: 1m

inhibit_rules:
  # Suppress warning alerts when a critical alert with the same service is firing
  - source_match:
      severity: critical
    target_match:
      severity: warning
    equal: ["service", "namespace"]

  # Suppress all pod-level alerts when a node is down
  - source_match:
      alertname: NodeDown
    target_match_re:
      alertname: "Pod.*"
    equal: ["node"]

  # Suppress deployment-level alerts during maintenance windows
  - source_match:
      alertname: MaintenanceWindow
    target_match_re:
      alertname: ".+"
    equal: ["namespace", "service"]

receivers:
  - name: "default-receiver"
    slack_configs:
      - channel: "#alerts-default"
        title: '[{{ .Status | toUpper }}] {{ .CommonLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

  - name: "pagerduty-critical"
    pagerduty_configs:
      - routing_key: "${PAGERDUTY_INTEGRATION_KEY}"
        severity: critical
        description: |
          {{ .CommonLabels.alertname }}: {{ .CommonAnnotations.summary }}
        details:
          firing: '{{ .Alerts.Firing | len }}'
          resolved: '{{ .Alerts.Resolved | len }}'
          runbook: '{{ .CommonAnnotations.runbook_url }}'
        links:
          - href: "{{ .CommonAnnotations.runbook_url }}"
            text: "Runbook"
          - href: "{{ .CommonAnnotations.dashboard_url }}"
            text: "Dashboard"

  - name: "pagerduty-payments"
    pagerduty_configs:
      - routing_key: "${PAGERDUTY_PAYMENTS_KEY}"
        severity: critical

  - name: "pagerduty-infrastructure"
    pagerduty_configs:
      - routing_key: "${PAGERDUTY_INFRA_KEY}"
        severity: critical

  - name: "slack-warnings"
    slack_configs:
      - api_url: "${SLACK_WEBHOOK_URL}"
        channel: "#alerts-warning"
        send_resolved: true
        color: '{{ if eq .Status "firing" }}warning{{ else }}good{{ end }}'
        title: |
          [{{ .Status | toUpper }}{{ if eq .Status "firing" }} - {{ .Alerts.Firing | len }}{{ end }}] {{ .CommonLabels.alertname }}
        title_link: "{{ .CommonAnnotations.dashboard_url }}"
        text: |
          *Summary:* {{ .CommonAnnotations.summary }}
          *Description:* {{ .CommonAnnotations.description }}
          *Runbook:* {{ .CommonAnnotations.runbook_url }}
          {{ range .Alerts -}}
          *Labels:*
          {{ range .Labels.SortedPairs }} • {{ .Name }}: `{{ .Value }}`
          {{ end }}
          {{ end }}
        footer: "Alertmanager | {{ .CommonLabels.job }}"

  - name: "slack-info"
    slack_configs:
      - channel: "#alerts-info"
        send_resolved: true
        color: good
        title: "[{{ .Status | toUpper }}] {{ .CommonLabels.alertname }}"
        text: "{{ .CommonAnnotations.description }}"

  - name: "email-oncall"
    email_configs:
      - to: "oncall@example.com"
        send_resolved: true
        html: |
          <h2>{{ .CommonLabels.alertname }}</h2>
          <p><strong>Status:</strong> {{ .Status }}</p>
          <p><strong>Summary:</strong> {{ .CommonAnnotations.summary }}</p>
          <p><strong>Description:</strong> {{ .CommonAnnotations.description }}</p>
          <p><a href="{{ .CommonAnnotations.runbook_url }}">Runbook</a> |
             <a href="{{ .CommonAnnotations.dashboard_url }}">Dashboard</a></p>
        headers:
          subject: '[{{ .Status | toUpper }}] {{ .CommonLabels.alertname }}: {{ .CommonAnnotations.summary }}'

  - name: "opsgenie-critical"
    opsgenie_configs:
      - api_key: "${OPSGENIE_API_KEY}"
        message: "{{ .CommonLabels.alertname }}: {{ .CommonAnnotations.summary }}"
        priority: P1
        details:
          description: "{{ .CommonAnnotations.description }}"
          runbook: "{{ .CommonAnnotations.runbook_url }}"
        tags: "{{ range .CommonLabels.SortedPairs }}{{ .Name }}={{ .Value }},{{ end }}"

  - name: "deadmanssnitch"
    webhook_configs:
      - url: "${DEADMANSSNITCH_URL}"
        send_resolved: false
```

### Alertmanager Deployment

**Docker:**
```bash
docker run -d \
  --name alertmanager \
  -p 9093:9093 \
  -v $(pwd)/alertmanager.yml:/etc/alertmanager/alertmanager.yml \
  -v $(pwd)/templates:/etc/alertmanager/templates \
  prom/alertmanager:v0.27.0 \
  --config.file=/etc/alertmanager/alertmanager.yml \
  --cluster.advertise-address=0.0.0.0:9093
```

**Kubernetes:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alertmanager
  namespace: monitoring
spec:
  replicas: 3   # HA cluster
  selector:
    matchLabels:
      app: alertmanager
  template:
    metadata:
      labels:
        app: alertmanager
    spec:
      containers:
        - name: alertmanager
          image: prom/alertmanager:v0.27.0
          args:
            - "--config.file=/etc/alertmanager/alertmanager.yml"
            - "--storage.path=/alertmanager"
            - "--cluster.peer=alertmanager-0.alertmanager:9094"
            - "--cluster.peer=alertmanager-1.alertmanager:9094"
            - "--cluster.peer=alertmanager-2.alertmanager:9094"
          ports:
            - containerPort: 9093
            - containerPort: 9094  # Cluster gossip port
          volumeMounts:
            - name: config
              mountPath: /etc/alertmanager
            - name: storage
              mountPath: /alertmanager
      volumes:
        - name: config
          secret:
            secretName: alertmanager-config
        - name: storage
          emptyDir: {}
```

---

## Alert Routing Tree

### Routing ASCII Diagram

```
Alert Routing Tree
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  All incoming alerts
          │
          ▼
  ┌───────────────────────────────────────────────────┐
  │              ROOT ROUTE                            │
  │  group_by: [alertname, service]                   │
  │  receiver: default-receiver (slack #alerts)       │
  └───────────────────────────┬───────────────────────┘
                              │
         ┌────────────────────┼──────────────────────┐
         │                    │                      │
         ▼                    ▼                      ▼
  ┌─────────────┐     ┌──────────────┐     ┌──────────────────┐
  │severity:    │     │severity:     │     │alertname:        │
  │ critical    │     │ warning      │     │ Watchdog         │
  └──────┬──────┘     └──────┬───────┘     └────────┬─────────┘
         │                   │                      │
         │              slack #alerts-warning       deadmanssnitch
         │
    ┌────┴──────────────────┐
    │                       │
    ▼                       ▼
 ┌────────────┐      ┌────────────────┐
 │ team:      │      │ team:          │
 │ payments   │      │ infrastructure │
 └──────┬─────┘      └───────┬────────┘
        │                    │
  PagerDuty             PagerDuty
  (payments key)        (infra key)
  +Slack #payments      +Slack #infra
```

### Team-Based Routing Configuration

```yaml
route:
  receiver: default-slack
  group_by: [alertname, namespace]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 12h

  routes:
    - match:
        severity: critical
      receiver: pagerduty-default
      routes:
        - match:
            team: frontend
          receiver: pagerduty-frontend
          routes:
            - match:
                env: staging
              receiver: slack-frontend  # Staging critical = Slack only
        - match:
            team: backend
          receiver: pagerduty-backend
        - match:
            team: data
          receiver: pagerduty-data
        - match:
            team: infrastructure
          receiver: pagerduty-infra

    - match_re:
        team: frontend|backend|data
      match:
        severity: warning
      receiver: slack-engineering
      group_wait: 2m

    - match:
        team: infrastructure
        severity: warning
      receiver: slack-infrastructure
```

---

## Alertmanager Inhibition

**Inhibition** suppresses alerts that are symptoms of an already-known problem. Without inhibition, a single root cause can generate dozens of noisy alerts.

**Why it matters:**

```
WITHOUT inhibition — node failure generates:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  1. NodeDown (1 alert — the actual problem)
  2. PodCrashLooping × 15 (pods can't start — not actually crashing)
  3. ServiceUnavailable × 8 (services unavailable because pods are down)
  4. HighErrorRate × 5 (errors because services are unavailable)
  5. HighLatency × 3 (timeouts because services are unavailable)
  6. InstanceDown × 15 (Prometheus targets unreachable)
  = 47 alerts fired for 1 root cause

WITH inhibition:
  1. NodeDown (fires)
  2. All pod/service/target alerts on that node = SUPPRESSED
  = 1 alert for 1 root cause
```

### Inhibition Rules YAML

```yaml
inhibit_rules:
  # When a node is down, suppress pod and target alerts on that node
  - source_match:
      alertname: NodeDown
    target_match_re:
      alertname: "(PodCrashLooping|ServiceUnavailable|InstanceDown|HighErrorRate)"
    equal: ["node"]

  # When a critical alert fires, suppress its warning counterpart
  - source_match:
      severity: critical
    target_match:
      severity: warning
    equal: ["alertname", "service", "namespace"]

  # When maintenance window is active, suppress all alerts for that service
  - source_match:
      alertname: MaintenanceWindow
      active: "true"
    target_match_re:
      alertname: ".+"
    equal: ["service", "namespace"]

  # When the whole cluster is down, suppress individual service alerts
  - source_match:
      alertname: ClusterDown
    target_match_re:
      alertname: ".*"
    equal: ["cluster"]

  # When deployment is in progress, suppress pod restart alerts
  - source_match:
      alertname: DeploymentInProgress
    target_match:
      alertname: PodCrashLooping
    equal: ["deployment", "namespace"]
```

---

## Alertmanager Silences

**Silences** temporarily suppress notifications for matching alerts. Unlike inhibition (which is automatic based on rules), silences are manually created — typically during planned maintenance, known issues, or debugging sessions.

### Creating Silences with `amtool`

```bash
# Install amtool
go install github.com/prometheus/alertmanager/cmd/amtool@latest

# Set Alertmanager URL
export ALERTMANAGER_URL=http://alertmanager:9093

# Create a silence for all alerts on a specific node during maintenance
amtool silence add \
  --alertmanager.url=http://alertmanager:9093 \
  --author="ops-team" \
  --comment="Planned maintenance: replacing disk on worker-01" \
  --duration=2h \
  "instance=~worker-01.*"

# Create a silence for a specific alert during a deployment
amtool silence add \
  alertname=PodCrashLooping \
  namespace=production \
  deployment=payment-service \
  --duration=30m \
  --comment="Deployment v1.2.3 in progress — expected pod restarts"

# List active silences
amtool silence query

# Expire a silence by ID
amtool silence expire abc12345-6789-abcd-ef01-234567890abc
```

### Silence via API

```bash
# Create silence via Alertmanager API
curl -X POST http://alertmanager:9093/api/v2/silences \
  -H 'Content-Type: application/json' \
  -d '{
    "matchers": [
      {
        "name": "alertname",
        "value": "DiskSpaceRunningLow",
        "isRegex": false
      },
      {
        "name": "instance",
        "value": "worker-01:9100",
        "isRegex": false
      }
    ],
    "startsAt": "2025-01-01T02:00:00.000Z",
    "endsAt":   "2025-01-01T06:00:00.000Z",
    "createdBy": "ops-team",
    "comment": "Known disk issue, replacement ordered"
  }'
```

---

## PagerDuty Integration

### Complete Receiver Configuration

```yaml
receivers:
  - name: "pagerduty-critical"
    pagerduty_configs:
      # v2 Events API integration key (recommended over API key)
      - routing_key: "${PAGERDUTY_ROUTING_KEY}"

        # Severity mapping: critical → P1, warning → P3
        severity: '{{ if eq .CommonLabels.severity "critical" }}critical{{ else }}warning{{ end }}'

        # Event summary (appears in PagerDuty incident title)
        description: "{{ .CommonLabels.alertname }}: {{ .CommonAnnotations.summary }}"

        # Rich details in the PagerDuty incident
        details:
          num_firing: "{{ .Alerts.Firing | len }}"
          num_resolved: "{{ .Alerts.Resolved | len }}"
          description: "{{ .CommonAnnotations.description }}"
          runbook_url: "{{ .CommonAnnotations.runbook_url }}"
          dashboard_url: "{{ .CommonAnnotations.dashboard_url }}"
          service: "{{ .CommonLabels.service }}"
          namespace: "{{ .CommonLabels.namespace }}"
          environment: "{{ .CommonLabels.env }}"

        # Links shown in PagerDuty interface
        links:
          - href: "{{ .CommonAnnotations.runbook_url }}"
            text: "📖 Runbook"
          - href: "{{ .CommonAnnotations.dashboard_url }}"
            text: "📊 Dashboard"

        # Images can be added (requires publicly accessible URL)
        images:
          - src: "https://grafana.example.com/render/d/abc?orgId=1"
            alt: "Current dashboard state"

        # Client information
        client: "Alertmanager"
        client_url: "http://alertmanager:9093"
```

### Required PagerDuty Settings

| Setting | Where to Find | Description |
|---------|--------------|-------------|
| `routing_key` | PagerDuty → Services → Integrations → Events API v2 | Per-service routing key |
| Service name | PagerDuty → Services | Logical service this integration alerts on |
| Escalation policy | PagerDuty → Escalation Policies | Who gets paged and in what order |

### Routing Example

```yaml
route:
  routes:
    - match:
        severity: critical
        team: payments
      receiver: pagerduty-payments
      continue: false  # Stop processing more routes once matched

    - match:
        severity: critical
        team: infrastructure
      receiver: pagerduty-infrastructure
      continue: false

    - match:
        severity: critical
      receiver: pagerduty-default  # Catch-all for critical
```

---

## OpsGenie Integration

### Complete Receiver Configuration

```yaml
receivers:
  - name: "opsgenie-critical"
    opsgenie_configs:
      - api_key: "${OPSGENIE_API_KEY}"
        api_url: "https://api.opsgenie.com/"

        # Alert message (appears as alert title in OpsGenie)
        message: "[{{ .Status | toUpper }}] {{ .CommonLabels.alertname }}: {{ .CommonAnnotations.summary }}"

        # Priority: P1 (critical), P2 (high), P3 (moderate), P4 (low), P5 (informational)
        priority: '{{ if eq .CommonLabels.severity "critical" }}P1{{ else if eq .CommonLabels.severity "warning" }}P3{{ else }}P5{{ end }}'

        # Team to assign the alert to
        teams: "{{ .CommonLabels.team }}"

        # Description shown in OpsGenie
        description: |
          {{ .CommonAnnotations.description }}
          Runbook: {{ .CommonAnnotations.runbook_url }}

        # Source of the alert
        source: "Alertmanager"

        # Tags for filtering in OpsGenie
        tags: |
          {{ range .CommonLabels.SortedPairs -}}
          {{ .Name }}:{{ .Value }},
          {{- end }}

        # Additional details
        details:
          'Service': '{{ .CommonLabels.service }}'
          'Environment': '{{ .CommonLabels.env }}'
          'Namespace': '{{ .CommonLabels.namespace }}'
          'Dashboard': '{{ .CommonAnnotations.dashboard_url }}'
          'Runbook': '{{ .CommonAnnotations.runbook_url }}'

        # Responder list (optional override)
        responders:
          - id: "${OPSGENIE_TEAM_ID}"
            type: team
```

### Required OpsGenie Settings

| Setting | Description |
|---------|-------------|
| `api_key` | Found in OpsGenie → Settings → API key management |
| Team name | Must match an existing OpsGenie team exactly |
| Escalation policy | Configured in OpsGenie, not in Alertmanager |
| Notification rules | Configured per user in OpsGenie settings |

---

## Slack Notifications

### Complete Slack Receiver Configuration

```yaml
receivers:
  - name: "slack-critical"
    slack_configs:
      - api_url: "${SLACK_CRITICAL_WEBHOOK}"
        channel: "#alerts-critical"
        send_resolved: true

        # Color: red for firing, green for resolved
        color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'

        # Notification title
        title: |
          {{ if eq .Status "firing" -}}
            🔴 [FIRING: {{ .Alerts.Firing | len }}] {{ .CommonLabels.alertname }}
          {{- else -}}
            ✅ [RESOLVED] {{ .CommonLabels.alertname }}
          {{- end }}

        title_link: "{{ .CommonAnnotations.dashboard_url }}"

        # Notification body
        text: |
          *Summary:* {{ .CommonAnnotations.summary }}

          *Description:*
          {{ .CommonAnnotations.description }}

          *Service:* `{{ .CommonLabels.service }}`
          *Namespace:* `{{ .CommonLabels.namespace }}`
          *Environment:* `{{ .CommonLabels.env }}`

          📖 <{{ .CommonAnnotations.runbook_url }}|Runbook>  |  📊 <{{ .CommonAnnotations.dashboard_url }}|Dashboard>

        # Footer shown at bottom of attachment
        footer: "Alertmanager • {{ .CommonLabels.cluster }}"
        footer_icon: "https://prometheus.io/assets/favicons/favicon.ico"

        # Mention on-call person when critical
        pretext: '{{ if eq .Status "firing" }}<!here> On-call: please investigate.{{ end }}'
```

### Slack Template File (`slack.tmpl`)

```
{{/* slack.tmpl — custom Slack alert templates */}}

{{ define "slack.title" -}}
  {{ if eq .Status "firing" -}}
    {{ if eq (len .Alerts.Firing) 1 -}}
      🔴 {{ (index .Alerts.Firing 0).Labels.alertname }}
    {{- else -}}
      🔴 [{{ len .Alerts.Firing }} firing] {{ .CommonLabels.alertname }}
    {{- end }}
  {{- else -}}
    ✅ [RESOLVED] {{ .CommonLabels.alertname }}
  {{- end }}
{{- end }}

{{ define "slack.color" -}}
  {{ if eq .Status "firing" -}}
    {{ if eq .CommonLabels.severity "critical" }}danger
    {{- else if eq .CommonLabels.severity "warning" }}warning
    {{- else }}#439FE0
    {{- end -}}
  {{- else }}good
  {{- end }}
{{- end }}

{{ define "slack.text" -}}
  {{ if eq .Status "firing" -}}
*Alert:* {{ .CommonLabels.alertname }} - `{{ .CommonLabels.severity }}`

*Summary:* {{ .CommonAnnotations.summary }}

*Description:*
{{ .CommonAnnotations.description }}

*Details:*
{{ range .CommonLabels.SortedPairs -}}
  • *{{ .Name }}:* `{{ .Value }}`
{{ end -}}

*Firing alerts:*
{{ range .Alerts.Firing -}}
  • <{{ .GeneratorURL }}|{{ .Labels.alertname }}> started at {{ .StartsAt | since }}
{{ end -}}

📖 <{{ .CommonAnnotations.runbook_url }}|Runbook> | 📊 <{{ .CommonAnnotations.dashboard_url }}|Dashboard>
  {{ else -}}
*Resolved:* {{ .CommonLabels.alertname }} has been resolved.
All {{ len .Alerts.Resolved }} alert(s) cleared after {{ (index .Alerts.Resolved 0).EndsAt | since }}.
  {{- end }}
{{- end }}
```

### How Alerts Look in Slack

```
┌─────────────────────────────────────────────────────────────────┐
│ 🔴 [FIRING: 2] HighErrorRate                                     │
│ https://grafana.example.com/d/abc?var-service=payment-service   │
├─────────────────────────────────────────────────────────────────┤
│ *Summary:* High HTTP error rate on payment-service              │
│                                                                  │
│ *Description:*                                                   │
│ Service payment-service is returning 5.23% error responses      │
│ over the last 5 minutes. Threshold is 1%.                       │
│                                                                  │
│ *Service:* `payment-service`                                     │
│ *Namespace:* `production`                                        │
│ *Environment:* `prod`                                            │
│                                                                  │
│ 📖 Runbook  |  📊 Dashboard                                      │
│                                                              ⚠️  │
│ Alertmanager • prod-cluster                    [10 mins ago]     │
└─────────────────────────────────────────────────────────────────┘
```

---

## Runbooks

### What is a Runbook?

A **runbook** (also called a "playbook") is a documented set of procedures for responding to a specific alert. It answers:
1. What does this alert mean?
2. What is the immediate impact on users?
3. What steps should I take to diagnose the issue?
4. What are the common causes and their fixes?
5. How do I escalate if I can't resolve it?

A good runbook reduces the mean time to resolution (MTTR) by enabling any on-call engineer — including someone unfamiliar with the service — to respond effectively.

### Runbook Template

```markdown
# Runbook: {AlertName}

**Alert:** `{alertname}`
**Severity:** critical / warning
**Team:** {team}
**Service:** {service}
**Last updated:** {date}
**Owner:** {name/team}

## Summary
One or two sentences describing what this alert means and why it matters.

## Impact
- **User-facing:** Describe what users experience when this alert fires.
- **Business:** Quantify the impact if possible (e.g., "Checkout is unavailable for X% of users").
- **SLO:** Which SLO is affected?

## Diagnosis Steps

### Step 1: Verify the alert is real
```bash
# Check current error rate
kubectl top pods -n production
curl -s http://prometheus:9090/api/v1/query?query=rate(http_requests_total{status=~"5.."}[5m])
```

### Step 2: Check service logs
```bash
kubectl logs -n production -l app=payment-service --tail=100 | grep ERROR
```

### Step 3: Check dependencies
- [ ] Is the database reachable?
- [ ] Is the upstream payment processor API responding?
- [ ] Are there any recent deployments?

## Common Causes

| Cause | Symptoms | Resolution |
|-------|---------|------------|
| Database connection pool exhausted | DB errors in logs | Increase pool size or restart service |
| Upstream API rate limiting | 429 errors in logs | Add retry with backoff, reduce request rate |
| Recent bad deployment | Errors started after deploy | Rollback: `kubectl rollout undo deployment/payment-service` |
| Certificate expired | TLS errors in logs | Rotate certificate (see cert runbook) |

## Escalation
- **Tier 1 (on-call engineer):** Follow diagnosis steps above. Expected resolution time: 30 minutes.
- **Tier 2:** `#payments-team` Slack channel + ping @payments-lead
- **Tier 3 (P0/P1 only):** Page payments team lead via PagerDuty escalation policy

## External Dependencies
- Stripe API status: https://status.stripe.com
- Internal DB status: https://internal-status.example.com

## References
- Service architecture: https://wiki.example.com/payment-service
- Grafana dashboard: https://grafana.example.com/d/payments
- Previous incidents: https://incidents.example.com?service=payments
```

### Complete Runbook: HighErrorRate

```markdown
# Runbook: HighErrorRate

**Alert:** `HighErrorRate`
**Severity:** critical
**Service:** Any HTTP service
**Owner:** Platform Engineering

## Summary
The HTTP error rate (5xx responses) for a service has exceeded 1% over a 5-minute window.
This means real users are receiving error responses right now.

## Impact
- Users receive error pages or failed API calls.
- If error rate > 5%, the service may be completely unavailable.
- SLO breach: availability SLO of 99.9% may be violated within hours.

## Diagnosis Steps

### 1. How bad is it?
```bash
# Get current error rate by endpoint
kubectl exec -n monitoring prometheus-0 -- \
  promtool query instant 'sum(rate(http_requests_total{status=~"5..",service="YOURSERVICE"}[5m])) by (status, path)'

# Get error count in last 5 minutes
kubectl logs -n production -l app=YOURSERVICE --since=5m | grep -c "ERROR\|FATAL"
```

### 2. Identify which endpoint is failing
```bash
# Top erroring endpoints
kubectl exec -n monitoring prometheus-0 -- \
  promtool query instant \
  'topk(10, rate(http_requests_total{status=~"5..",service="YOURSERVICE"}[5m]))'
```

### 3. Check for recent deployments
```bash
kubectl rollout history deployment/YOURSERVICE -n production
kubectl describe deployment/YOURSERVICE -n production | grep "Updated\|Image"
```

### 4. Check service logs
```bash
kubectl logs -n production -l app=YOURSERVICE --tail=200 | grep -E "ERROR|FATAL|panic"
# Previous pod (if pod restarted)
kubectl logs -n production -l app=YOURSERVICE --previous --tail=200
```

### 5. Check dependencies
```bash
# Database health
kubectl exec -n production deploy/YOURSERVICE -- \
  nc -zv db.production.svc.cluster.local 5432

# Redis health
kubectl exec -n production deploy/YOURSERVICE -- \
  redis-cli -h redis.production.svc.cluster.local ping
```

## Resolution Actions

| Cause | Fix |
|-------|-----|
| Bad deployment | `kubectl rollout undo deployment/YOURSERVICE -n production` |
| DB connection issue | Restart service: `kubectl rollout restart deployment/YOURSERVICE -n production` |
| Memory exhaustion | Scale horizontally: `kubectl scale deployment/YOURSERVICE --replicas=N` |
| Upstream dependency down | Enable circuit breaker / fallback. Check upstream status page. |
| Config error | Review recent ConfigMap changes. Apply previous config. |

## Escalation Path
1. **On-call** resolves via steps above (SLA: 30 min)
2. If unresolved → Slack #payments-critical + page service owner
3. If unresolved after 1 hour → Incident Commander, P1 declared
```

---

## Dead Man's Switch / Watchdog Alerts

### What They Are and Why They're Needed

A **dead man's switch** (or watchdog alert) is an alert that should **always be firing**. If the alert stops firing, it means your monitoring pipeline itself has broken — Prometheus is down, Alertmanager is unreachable, or the notification channel is broken.

```
Why a Watchdog Alert is Critical
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Without watchdog:
  ┌────────────────────────────────────────────────────────┐
  │ Production is on fire           ▲                      │
  │                                 │                      │
  │ Prometheus is down             SILENCE                 │
  │                                 │                      │
  │ No alerts fire               ← You have NO IDEA        │
  │                                 │                      │
  │ You find out from users        ▼                       │
  └────────────────────────────────────────────────────────┘

  With watchdog:
  ┌────────────────────────────────────────────────────────┐
  │ Prometheus is down                                     │
  │    → Watchdog stops firing                            │
  │    → Dead Man's Snitch API stops receiving heartbeat  │
  │    → Dead Man's Snitch sends you an alert             │
  │                                                        │
  │ "Your monitoring stopped monitoring. Fix it NOW."     │
  └────────────────────────────────────────────────────────┘
```

### Prometheus Watchdog Alert Rule

```yaml
groups:
  - name: watchdog
    rules:
      - alert: Watchdog
        expr: vector(1)
        labels:
          severity: none
        annotations:
          summary: "Watchdog — this alert must always be firing"
          description: >
            This is a watchdog/dead man's switch alert. It is designed to always
            be in firing state. If this alert stops firing, it means Prometheus
            or Alertmanager is down and the monitoring pipeline has broken.
            This alert should route to a dead man's snitch service (e.g.,
            Dead Man's Snitch, healthchecks.io, PagerDuty Watchdog) that pages
            when it stops receiving the heartbeat.
```

### Alertmanager Dead Man's Snitch Configuration

```yaml
route:
  routes:
    # Watchdog route — sends a heartbeat every minute
    - match:
        alertname: Watchdog
      receiver: deadmanssnitch
      group_wait: 0s
      group_interval: 1m    # Re-notify every 1 minute (the heartbeat)
      repeat_interval: 1m

receivers:
  - name: deadmanssnitch
    # Dead Man's Snitch (https://deadmanssnitch.com)
    webhook_configs:
      - url: "https://nosnch.in/${DEADMANSSNITCH_TOKEN}"
        send_resolved: false  # Only send firing — absence of signal = page
```

**Alternative: healthchecks.io**
```yaml
receivers:
  - name: healthchecks-watchdog
    webhook_configs:
      - url: "https://hc-ping.com/${HEALTHCHECKS_UUID}"
        send_resolved: false
```

---

## Multi-Window Multi-Burn-Rate Alerting for SLOs

Multi-window multi-burn-rate alerting detects error budget exhaustion at different time scales — fast burns (imminent exhaustion) get immediate pages; slow burns get warnings.

### The Concept

```
Burn Rate Explained
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  For a 99.9% SLO (30-day period):
  • Error budget = 0.1% = 43.8 minutes/month

  Burn rate 1x  → consuming budget at exactly the sustainable rate
                  (full budget exhausted in exactly 30 days)

  Burn rate 14.4x → budget exhausted in 30 days / 14.4 = 2 hours
                    → SHORT WINDOW: alert immediately

  Burn rate 6x  → budget exhausted in 5 days
                  → MEDIUM WINDOW: page with some urgency

  Burn rate 1x  → budget will last full month
                  → LONG WINDOW: create a ticket

  Why multiple windows?
  • Short window (1h) only: misses slow burns that are just as damaging
  • Long window (6h+) only: too slow to catch fast burns before damage is done
  • BOTH: catches fast burns immediately AND slow burns before month-end
```

### Complete Alerting Rules YAML (Four Windows)

```yaml
groups:
  - name: slo.rules
    rules:
      # ─────────────────────────────────────────────────────────────────────
      # RECORDING RULES — pre-compute error rates for efficiency
      # ─────────────────────────────────────────────────────────────────────
      - record: job:slo_errors:rate5m
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) by (job)
          /
          sum(rate(http_requests_total[5m])) by (job)

      - record: job:slo_errors:rate30m
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[30m])) by (job)
          /
          sum(rate(http_requests_total[30m])) by (job)

      - record: job:slo_errors:rate1h
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[1h])) by (job)
          /
          sum(rate(http_requests_total[1h])) by (job)

      - record: job:slo_errors:rate6h
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[6h])) by (job)
          /
          sum(rate(http_requests_total[6h])) by (job)

      - record: job:slo_errors:rate3d
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[3d])) by (job)
          /
          sum(rate(http_requests_total[3d])) by (job)

  - name: slo.alerting
    rules:
      # ─────────────────────────────────────────────────────────────────────
      # WINDOW 1: Fast burn — budget exhausted in ~2 hours
      # Burn rate: 14.4x for 99.9% SLO
      # (1 - 0.999) * 14.4 = 0.0144
      # Short window: 5m | Long window: 1h
      # Severity: CRITICAL — page immediately
      # ─────────────────────────────────────────────────────────────────────
      - alert: SLOBurnRateCritical
        expr: |
          job:slo_errors:rate5m > (14.4 * (1 - 0.999))
          AND
          job:slo_errors:rate1h > (14.4 * (1 - 0.999))
        for: 2m
        labels:
          severity: critical
          slo_window: "5m_1h"
          burn_rate: "14.4x"
        annotations:
          summary: "Critical SLO burn rate on {{ $labels.job }}"
          description: >
            Error budget is burning at 14.4x the sustainable rate.
            At this rate, the 30-day error budget will be exhausted
            in approximately 2 hours. Current error rate: {{ printf "%.4f" (mul $value 100) }}%.
            Immediate action required.
          runbook_url: "https://wiki.example.com/runbooks/slo-burn-rate"

      # ─────────────────────────────────────────────────────────────────────
      # WINDOW 2: Fast-medium burn — budget exhausted in ~5 hours
      # Burn rate: 6x for 99.9% SLO
      # Short window: 30m | Long window: 6h
      # Severity: CRITICAL — page soon
      # ─────────────────────────────────────────────────────────────────────
      - alert: SLOBurnRateHigh
        expr: |
          job:slo_errors:rate30m > (6 * (1 - 0.999))
          AND
          job:slo_errors:rate6h > (6 * (1 - 0.999))
        for: 15m
        labels:
          severity: critical
          slo_window: "30m_6h"
          burn_rate: "6x"
        annotations:
          summary: "High SLO burn rate on {{ $labels.job }}"
          description: >
            Error budget is burning at 6x the sustainable rate.
            The 30-day budget will be exhausted in approximately 5 hours.
            Current error rate: {{ printf "%.4f" (mul $value 100) }}%.
            Investigation required urgently.
          runbook_url: "https://wiki.example.com/runbooks/slo-burn-rate"

      # ─────────────────────────────────────────────────────────────────────
      # WINDOW 3: Medium burn — budget exhausted in ~2.5 days
      # Burn rate: 3x for 99.9% SLO
      # Short window: 6h | Long window: 3d
      # Severity: WARNING — investigate during business hours
      # ─────────────────────────────────────────────────────────────────────
      - alert: SLOBurnRateWarning
        expr: |
          job:slo_errors:rate6h > (3 * (1 - 0.999))
          AND
          job:slo_errors:rate3d > (3 * (1 - 0.999))
        for: 1h
        labels:
          severity: warning
          slo_window: "6h_3d"
          burn_rate: "3x"
        annotations:
          summary: "Elevated SLO burn rate on {{ $labels.job }}"
          description: >
            Error budget is burning at 3x the sustainable rate.
            The 30-day budget will be exhausted in approximately 2.5 days
            if the current error rate continues.
            Current error rate: {{ printf "%.4f" (mul $value 100) }}%.
            Investigate during next business hours.
          runbook_url: "https://wiki.example.com/runbooks/slo-burn-rate"

      # ─────────────────────────────────────────────────────────────────────
      # WINDOW 4: Slow burn — budget on track to exhaust by month-end
      # Burn rate: 1x for 99.9% SLO  
      # Long window: 3d
      # Severity: INFO — ticket for next sprint
      # ─────────────────────────────────────────────────────────────────────
      - alert: SLOBurnRateInfo
        expr: |
          job:slo_errors:rate3d > (1 * (1 - 0.999))
        for: 6h
        labels:
          severity: info
          slo_window: "3d"
          burn_rate: "1x"
        annotations:
          summary: "SLO burn rate at sustainable limit for {{ $labels.job }}"
          description: >
            Error budget is being consumed at or above the sustainable rate.
            If this continues for the month, the budget will be exactly
            exhausted by month-end. No immediate action required, but
            a ticket should be raised to investigate the error source.
            Current error rate: {{ printf "%.4f" (mul $value 100) }}%.
```

**Burn rate to window mapping:**

| Alert | Short Window | Long Window | Burn Rate | Time to Exhaustion | Severity |
|-------|-------------|------------|-----------|-------------------|----------|
| `SLOBurnRateCritical` | 5m | 1h | 14.4x | ~2 hours | critical |
| `SLOBurnRateHigh` | 30m | 6h | 6x | ~5 hours | critical |
| `SLOBurnRateWarning` | 6h | 3d | 3x | ~2.5 days | warning |
| `SLOBurnRateInfo` | 3d | — | 1x | ~30 days | info |

---

## Next Steps

1. **Audit existing alerts** — Review all current alerts using the "pages at 3am" test. Delete or demote any that aren't immediately actionable.
2. **Implement the symptom-based approach** — For each cause-based alert, identify the corresponding user-visible symptom and alert on that instead.
3. **Deploy Alertmanager with HA** — Run at least 3 replicas with `--cluster.peer` configured for gossip-based deduplication.
4. **Configure inhibition rules** — Start with the node-down inhibition pattern to dramatically reduce alert noise.
5. **Create runbooks** — For every critical alert, create a runbook following the template in this guide. Link it in `annotations.runbook_url`.
6. **Implement multi-window burn-rate alerting** — Replace simple threshold alerts for your SLOs with the four-window burn rate approach.
7. **Set up a Dead Man's Switch** — Deploy the Watchdog alert + Dead Man's Snitch to catch silent monitoring failures.
8. **Track alert quality metrics** — Measure the ratio of actionable alerts to total alerts. Target: > 80% of pages require action.

**Related guides in this series:**
- [01-METRICS.md](01-METRICS.md) — Prometheus metrics and PromQL for writing alert expressions
- [07-SLIS-SLOS-SLAS.md](07-SLIS-SLOS-SLAS.md) — Error budgets and SLO-based alerting

---

## Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0.0 | 2025-01-01 | Platform Engineering | Initial document |
| 1.1.0 | 2025-01-20 | Platform Engineering | Added multi-window burn rate alerting section |
| 1.2.0 | 2025-02-10 | Platform Engineering | Added OpsGenie integration, expanded runbook template |
| 1.3.0 | 2025-03-05 | Platform Engineering | Added Slack template file and alert appearance example |
| 1.4.0 | 2025-04-01 | Platform Engineering | Expanded inhibition rules section with cascade examples |
| 1.5.0 | 2025-06-01 | Platform Engineering | Added Dead Man's Switch section |
