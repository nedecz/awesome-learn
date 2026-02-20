# SLIs, SLOs, and SLAs

## Table of Contents

1. [Definitions and Differences](#definitions-and-differences)
2. [Error Budget](#error-budget)
3. [Setting Meaningful SLOs](#setting-meaningful-slos)
4. [Common SLI Types](#common-sli-types)
5. [SLO Tracking with Prometheus and Grafana](#slo-tracking-with-prometheus-and-grafana)
6. [PromQL Examples for SLIs](#promql-examples-for-slis)
7. [Multi-Window Multi-Burn-Rate Alerting](#multi-window-multi-burn-rate-alerting)
8. [Error Budget Burn Rate](#error-budget-burn-rate)
9. [SLA Negotiation Considerations](#sla-negotiation-considerations)
10. [Example SLOs](#example-slos)
11. [Google SRE Approach to SLOs](#google-sre-approach-to-slos)
12. [Next Steps](#next-steps)
13. [Version History](#version-history)

---

## Definitions and Differences

### SLI — Service Level Indicator

A **Service Level Indicator** is a **quantitative measure of some aspect of the level of service being provided**. It is a ratio: the proportion of measurements that were "good" out of all measurements. An SLI is a fact — it tells you what actually happened.

> "The fraction of HTTP requests that returned a successful response in under 200ms."

An SLI must be:
- **Measurable**: computable from real production data (logs, metrics, traces)
- **Meaningful**: reflects something users actually care about
- **Actionable**: something the service team can influence

### SLO — Service Level Objective

A **Service Level Objective** is a **target value or range for an SLI**. It expresses what level of service you intend to provide. An SLO is a goal.

> "99.9% of HTTP requests will return a successful response in under 200ms over a 30-day rolling window."

### SLA — Service Level Agreement

A **Service Level Agreement** is a **contractual commitment** (formal or informal) to meet an SLO, with defined **consequences for breach**. An SLA is a promise backed by real stakes.

> "If availability falls below 99.5% in any calendar month, affected customers receive a 10% service credit."

### Comparison Table

| Dimension | SLI | SLO | SLA |
|-----------|-----|-----|-----|
| **Definition** | Measured quantity of service quality | Target for the SLI | Contractual commitment with consequences |
| **Who sets it** | Service team (based on measurement) | Service team + stakeholders | Legal/business team + customers |
| **Consequence of breach** | No direct consequence (it's just data) | Internal action: stop feature work, focus on reliability | Legal penalty, financial credit, customer churn |
| **Tightness** | Reflects what's actually achievable | Slightly tighter than current performance | Looser than SLO (give yourself buffer) |
| **Example (web API)** | `p99 latency = 180ms over last 30d` | `p99 latency < 200ms` (99.9% of requests) | `p99 latency < 500ms; breach = 10% credit` |
| **Audience** | Engineering team | Engineering + Product | Customers + Legal |
| **Frequency of review** | Real-time/continuous | Monthly | Annually or per contract |

### Hierarchy Diagram

```
SLI → SLO → SLA Hierarchy with Error Budget Layer
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌────────────────────────────────────────────────────────────────┐
  │  SLA  (contractual, externally-facing)                         │
  │  "99.5% availability per month or 10% credit"                  │
  │                                                                 │
  │  ┌──────────────────────────────────────────────────────────┐  │
  │  │  SLO  (internal target, guides engineering decisions)     │  │
  │  │  "99.9% availability per month"                          │  │
  │  │                                                           │  │
  │  │  ┌──────────────────────────────────────────────────┐   │  │
  │  │  │  ERROR BUDGET  (what's left between 100% and SLO)│   │  │
  │  │  │  100% - 99.9% = 0.1% = 43.8 min/month           │   │  │
  │  │  │                                                   │   │  │
  │  │  │  ┌────────────────────────────────────────────┐  │   │  │
  │  │  │  │  SLI (measured reality)                    │  │   │  │
  │  │  │  │  "good_requests / total_requests"          │  │   │  │
  │  │  │  │  Currently: 99.97% availability            │  │   │  │
  │  │  │  └────────────────────────────────────────────┘  │   │  │
  │  │  └──────────────────────────────────────────────────┘   │  │
  │  └──────────────────────────────────────────────────────────┘  │
  └────────────────────────────────────────────────────────────────┘

  Note: SLA target MUST be looser than SLO target.
  If SLO = 99.9%, SLA = 99.5% gives you a 0.4% buffer before
  customers are entitled to credits.
```

---

## Error Budget

### Concept Explanation

The **error budget** is the amount of unreliability you are permitted to have while still meeting your SLO. It is the inverse of the SLO expressed as an allowance:

```
Error Budget = 1 - SLO Target

For 99.9% SLO:
  Error Budget = 1 - 0.999 = 0.001 = 0.1%
  In a 30-day month: 0.001 × 30 × 24 × 60 = 43.2 minutes
```

The error budget has two critical uses:
1. **Innovation enabler**: As long as budget is healthy, teams can ship features and take risks (controlled experiments, deployments, maintenance)
2. **Reliability enforcer**: When budget is nearly exhausted, teams must stop feature work and focus on reliability

### Error Budget Calculation Table

| SLO Target | Monthly Allowed Downtime | Weekly Allowed | Daily Allowed | Seconds/Month |
|-----------|------------------------|----------------|---------------|---------------|
| 99.0% | 7h 18m | 1h 41m | 14m 24s | 26,280s |
| 99.5% | 3h 39m | 50m 48s | 7m 12s | 13,140s |
| 99.9% | 43m 49s | 10m 4s | 1m 26s | 2,629s |
| 99.95% | 21m 54s | 5m 2s | 43s | 1,314s |
| 99.99% | 4m 23s | 1m 0s | 8.6s | 263s |
| 99.999% | 26s | 6s | 0.86s | 26s |

> **Rule of thumb**: Each additional "nine" reduces your error budget by ~10x.

### Error Budget Policy

The error budget policy defines what action to take based on the state of the budget:

| Budget State | Budget Remaining | Action |
|-------------|-----------------|--------|
| **Healthy** | > 50% remaining at midpoint of period | Proceed normally; feature development and deployments unrestricted |
| **Cautious** | 25–50% remaining | Increase deployment scrutiny; require canary deployments for risky changes |
| **Burning** | 10–25% remaining | Freeze non-critical deployments; focus engineering on reliability improvements; postmortem any recent incidents |
| **Exhausted** | < 10% remaining | Full deployment freeze (emergency fixes only); all engineers on reliability; exec notification; postmortem required |
| **Breached** | 0% or SLA violated | SLA breach handling: customer communication, financial credits, emergency response team |

```
Error Budget Over Time (30-day window)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  100% ├──────────────────────────────────────────────────────
       │
   75% │ HEALTHY — full steam ahead ──────────────────────────
       │                                         ╲
   50% │ CAUTIOUS — increase scrutiny             ╲──────────
       │                                              ╲
   25% │ BURNING — freeze deployments                  ╲──────
       │                                                 ╲
   10% │ EXHAUSTED — incident response                    ╲───
       │
    0% ├──────────────────────────────────────────────────────
        Day 1                                        Day 30

  Scenario A (dotted): Steady consumption rate → healthy all month
  Scenario B (solid): Incident on day 20 → budget exhausted → freeze
```

---

## Setting Meaningful SLOs

### Start with User Journeys, Not Service Boundaries

The most common SLO mistake is setting them at the service boundary rather than at the user journey level.

```
Wrong approach — SLO per service:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  order-service:    99.9% availability  ✓
  payment-service:  99.9% availability  ✓
  shipping-service: 99.9% availability  ✓
  
  BUT: P(all three available) = 0.999³ = 99.7% ← User actually gets THIS

Right approach — SLO per user journey:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  "User can complete checkout end-to-end": 99.9% success rate
  → Measures the thing the user actually cares about
  → Forces all three services to be reliable enough together
```

### The "What Does a User Care About?" Approach

For every SLO candidate, ask: **"When this breaks, does a user notice?"**

| User Journey | What they care about |
|--------------|---------------------|
| Log in | Can they log in? Is it fast? |
| Search | Do results appear? Are they correct? Fast? |
| Add to cart | Does the item appear in the cart? |
| Checkout | Does payment succeed? Is the order confirmed? |
| Load page | Does the page render? Under how many seconds? |
| Upload file | Does the upload succeed? How long does it take? |

### Starting Points — SLO Reference Table

| Service Type | Availability SLO | Latency SLO (p99) | Notes |
|-------------|-----------------|-------------------|-------|
| Interactive web API | 99.9% | < 500ms | Users abandon > 1s on interactive actions |
| Background job API | 99.5% | < 5s | Users tolerant of slightly longer waits |
| Real-time data feed | 99.95% | < 100ms | Low latency is core to the product value |
| Batch job | 99% success rate | Complete within SLA window | Measured by job completion, not HTTP latency |
| Data pipeline | 99.9% | Freshness < 1 hour | Freshness SLI matters most |
| Internal service (non-user-facing) | 99.5% | < 1s | Less strict; no direct user impact |
| Payment processing | 99.99% | < 2s | Revenue impact requires high bar |
| Auth/login | 99.99% | < 200ms | All services depend on this |

### How to Set Your First SLO

```
First SLO: The Four-Step Process
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Step 1: MEASURE
    Look at the last 30–90 days of data.
    "What is our current availability?"
    "What is our P99 latency right now?"

  Step 2: SET AT (OR SLIGHTLY ABOVE) CURRENT PERFORMANCE
    If you're currently at 99.7% availability → set SLO at 99.5%
    This ensures the SLO reflects achievable targets, not wishful thinking.
    You will tighten it over time.

  Step 3: DOCUMENT THE ERROR BUDGET POLICY
    What happens when you use 50% of budget?
    What happens when it's exhausted?

  Step 4: REVIEW IN 90 DAYS
    Is the SLO too loose (never alerting)? Tighten it.
    Is it too tight (constantly breaching)? Loosen it or invest in reliability.
```

### Common SLO Mistakes

| Mistake | Why it's wrong | Better approach |
|---------|---------------|----------------|
| Setting SLO at 100% | Impossible; any incident = breach; removes space for planned maintenance | Start at 99.9% or lower |
| Setting SLO without data | You don't know if it's achievable | Measure first, set later |
| Too many SLOs | Alert fatigue; hard to prioritize | 1–3 SLOs per critical user journey |
| SLOs for internal metrics | Users don't care about your CPU | SLIs must be user-visible |
| Never reviewing SLOs | SLOs become stale as service evolves | Quarterly review cycle |
| No error budget policy | Knowing the SLO is breaching but no defined action | Write the policy before you need it |

---

## Common SLI Types

### Overview Table

| SLI Type | Formula | When to Use | Example |
|----------|---------|-------------|---------|
| **Availability** | good_requests / total_requests | Always (for request-based services) | 200/201/202/204 responses / all responses |
| **Latency** | requests_under_threshold / total_requests | When speed is critical to UX | Requests < 500ms / all requests |
| **Throughput** | successful_items_processed / expected_items | Batch jobs, pipelines | Orders processed / orders received |
| **Error Rate** | error_requests / total_requests | Complement to availability | 5xx responses / all responses |
| **Freshness** | data_points_newer_than_threshold / all_data_points | Data pipelines, caches, feeds | Records updated < 1h ago / all records |
| **Correctness** | correct_responses / all_responses | Computation services, search | Responses matching ground truth / all |
| **Coverage** | items_processed / items_submitted | Batch, ETL, async processing | Files indexed / files uploaded |
| **Durability** | successfully_retrieved / successfully_stored | Storage services | Objects retrievable / objects written |

### Availability SLI

```
Availability SLI Formula:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Availability = good_requests / total_requests

  Where "good" typically means:
  - HTTP: status code not 5xx (i.e., not a server error)
    OR: status code is 2xx (stricter)
  - gRPC: status code OK
  - Background job: completed without error

  What to EXCLUDE from total_requests:
  - Requests from synthetic monitors / health checks (no real user impact)
  - Expected 4xx responses (client errors; not a service failure)
  - Requests to /favicon.ico, /robots.txt (noise)
```

### Latency SLI

```
Latency SLI — Use a threshold, not an average!
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  BAD:  avg(request_duration) < 200ms
        → A single 10-second request doesn't affect the average much
        → Users with slow requests are "hidden" in the average

  GOOD: requests_under_200ms / total_requests > 0.99
        → This is a ratio SLI: "99% of requests complete under 200ms"
        → Every slow request is directly counted against the SLI
        → Same form as availability SLI (fraction of good events)

  Implementation with Prometheus histograms:
  rate(http_request_duration_seconds_bucket{le="0.2"}[5m])
  / rate(http_request_duration_seconds_count[5m])
```

### Freshness SLI (for Data Pipelines)

```python
# Freshness SLI measures how current your data is

# Prometheus gauge — updated on each pipeline run
last_successful_pipeline_run_timestamp = Gauge(
    'pipeline_last_success_timestamp_seconds',
    'Unix timestamp of last successful pipeline completion',
    ['pipeline_name']
)

# SLI: fraction of time the data is "fresh" (updated within threshold)
# PromQL:
# time() - pipeline_last_success_timestamp_seconds < 3600
# (returns 1 if fresh, 0 if stale)
```

---

## SLO Tracking with Prometheus and Grafana

### Recording Rules for SLI Calculation

```yaml
# slo-recording-rules.yaml
groups:
  - name: slo.availability
    interval: 30s
    rules:
      # 5-minute window SLI
      - record: job:sli_availability:ratio_rate5m
        expr: |
          sum(rate(http_requests_total{status!~"5.."}[5m])) by (job)
          /
          sum(rate(http_requests_total[5m])) by (job)

      # 30-minute window SLI
      - record: job:sli_availability:ratio_rate30m
        expr: |
          sum(rate(http_requests_total{status!~"5.."}[30m])) by (job)
          /
          sum(rate(http_requests_total[30m])) by (job)

      # 1-hour window SLI
      - record: job:sli_availability:ratio_rate1h
        expr: |
          sum(rate(http_requests_total{status!~"5.."}[1h])) by (job)
          /
          sum(rate(http_requests_total[1h])) by (job)

      # 6-hour window SLI  
      - record: job:sli_availability:ratio_rate6h
        expr: |
          sum(rate(http_requests_total{status!~"5.."}[6h])) by (job)
          /
          sum(rate(http_requests_total[6h])) by (job)

      # 30-day window SLI (monthly compliance)
      - record: job:sli_availability:ratio_rate30d
        expr: |
          sum(rate(http_requests_total{status!~"5.."}[30d])) by (job)
          /
          sum(rate(http_requests_total[30d])) by (job)

  - name: slo.latency
    rules:
      # Latency SLI: fraction of requests under 500ms
      - record: job:sli_latency_500ms:ratio_rate5m
        expr: |
          sum(rate(http_request_duration_seconds_bucket{le="0.5"}[5m])) by (job)
          /
          sum(rate(http_request_duration_seconds_count[5m])) by (job)

      # 30-day latency SLI
      - record: job:sli_latency_500ms:ratio_rate30d
        expr: |
          sum(rate(http_request_duration_seconds_bucket{le="0.5"}[30d])) by (job)
          /
          sum(rate(http_request_duration_seconds_count[30d])) by (job)

  - name: slo.error_budget
    rules:
      # Error budget remaining (as fraction of total budget)
      # Assumes SLO target of 99.9% (error_budget = 0.001)
      - record: job:slo_error_budget_remaining:ratio_rate30d
        expr: |
          (
            job:sli_availability:ratio_rate30d - 0.999
          ) / (1 - 0.999)
        labels:
          slo: "availability-99.9"
```

### Grafana Panel Configuration

```json
{
  "title": "SLO Availability — 30-day rolling window",
  "type": "gauge",
  "options": {
    "reduceOptions": {"calcs": ["lastNotNull"]},
    "orientation": "auto",
    "showThresholdLabels": true,
    "showThresholdMarkers": true
  },
  "fieldConfig": {
    "defaults": {
      "unit": "percentunit",
      "min": 0.99,
      "max": 1.0,
      "thresholds": {
        "mode": "absolute",
        "steps": [
          {"color": "red",    "value": null},
          {"color": "yellow", "value": 0.999},
          {"color": "green",  "value": 0.9999}
        ]
      }
    }
  },
  "targets": [
    {
      "expr": "job:sli_availability:ratio_rate30d{job='payment-service'}",
      "legendFormat": "Availability SLI (30d)"
    }
  ]
}
```

### Error Budget Remaining Gauge

```json
{
  "title": "Error Budget Remaining",
  "type": "gauge",
  "targets": [
    {
      "expr": "clamp_min(job:slo_error_budget_remaining:ratio_rate30d{job='payment-service'}, -1)",
      "legendFormat": "Error Budget Remaining"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "unit": "percentunit",
      "thresholds": {
        "steps": [
          {"color": "red",    "value": null},
          {"color": "orange", "value": 0.1},
          {"color": "yellow", "value": 0.25},
          {"color": "green",  "value": 0.5}
        ]
      }
    }
  }
}
```

### Burn Rate Panel

```json
{
  "title": "Error Budget Burn Rate (1h)",
  "description": "1.0 = sustainable rate (budget lasts exactly 30 days). >1.0 = burning faster than sustainable.",
  "type": "timeseries",
  "targets": [
    {
      "expr": "(1 - job:sli_availability:ratio_rate1h{job='payment-service'}) / (1 - 0.999)",
      "legendFormat": "Burn Rate (1h)"
    },
    {
      "expr": "(1 - job:sli_availability:ratio_rate6h{job='payment-service'}) / (1 - 0.999)",
      "legendFormat": "Burn Rate (6h)"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "unit": "none",
      "custom": {
        "thresholdsStyle": {"mode": "line+area"}
      },
      "thresholds": {
        "steps": [
          {"color": "transparent", "value": null},
          {"color": "yellow",      "value": 6},
          {"color": "red",         "value": 14.4}
        ]
      }
    }
  }
}
```

---

## PromQL Examples for SLIs

```promql
# ── 1. AVAILABILITY SLI — fraction of successful requests ──────────────────
# Calculates the percentage of non-5xx responses over the last 30 days.
# This is your primary SLO compliance metric.

sum(rate(http_requests_total{job="payment-service", status!~"5.."}[30d]))
/
sum(rate(http_requests_total{job="payment-service"}[30d]))


# ── 2. ERROR RATE — 30-day window ──────────────────────────────────────────
# Inverse of availability SLI. Shows the fraction of requests that errored.

1 - (
  sum(rate(http_requests_total{job="payment-service", status!~"5.."}[30d]))
  /
  sum(rate(http_requests_total{job="payment-service"}[30d]))
)


# ── 3. LATENCY SLI — requests completing under threshold ───────────────────
# Fraction of requests completing under 500ms over 30 days.
# Uses histogram_bucket rather than histogram_quantile for a proper SLI ratio.

sum(rate(http_request_duration_seconds_bucket{job="payment-service", le="0.5"}[30d]))
/
sum(rate(http_request_duration_seconds_count{job="payment-service"}[30d]))


# ── 4. ERROR BUDGET REMAINING — percentage left ────────────────────────────
# How much of the 30-day error budget is remaining?
# Positive = budget remaining; Negative = SLO breached.
# Assumes SLO target = 99.9% (error_budget = 0.001)

(
  (
    sum(rate(http_requests_total{job="payment-service", status!~"5.."}[30d]))
    /
    sum(rate(http_requests_total{job="payment-service"}[30d]))
  )
  - 0.999
)
/ (1 - 0.999)


# ── 5. ERROR BUDGET CONSUMED TODAY ────────────────────────────────────────
# What fraction of the total monthly error budget was consumed in the last 24h?
# budget_rate = (monthly_budget / month_duration) per minute
# 30d in minutes = 43200

(
  sum(rate(http_requests_total{job="payment-service", status=~"5.."}[24h]))
  /
  sum(rate(http_requests_total{job="payment-service"}[24h]))
)
/ (1 - 0.999)
* (24 * 60)    # hours consumed as fraction of 30-day budget
/ (30 * 24 * 60)


# ── 6. BURN RATE — 1-hour window ───────────────────────────────────────────
# How fast is the error budget being consumed relative to sustainable rate?
# 1.0 = sustainable (budget lasts exactly 30 days at this rate)
# 14.4 = budget exhausted in 2 hours

(
  1 - (
    sum(rate(http_requests_total{job="payment-service", status!~"5.."}[1h]))
    /
    sum(rate(http_requests_total{job="payment-service"}[1h]))
  )
)
/ (1 - 0.999)


# ── 7. BURN RATE — 6-hour window ───────────────────────────────────────────
# Longer window; less sensitive to short spikes but catches sustained burns.

(
  1 - (
    sum(rate(http_requests_total{job="payment-service", status!~"5.."}[6h]))
    /
    sum(rate(http_requests_total{job="payment-service"}[6h]))
  )
)
/ (1 - 0.999)


# ── 8. MULTI-WINDOW BURN RATE CHECK ────────────────────────────────────────
# This is the exact expression used in multi-window burn rate alerts.
# Both short AND long window must be elevated to avoid false positives from spikes.
# Returns 1 (true) when both windows show high burn rate.

(
  (
    1 - (
      sum(rate(http_requests_total{job="payment-service", status!~"5.."}[5m]))
      /
      sum(rate(http_requests_total{job="payment-service"}[5m]))
    )
  ) / (1 - 0.999) > 14.4
)
AND
(
  (
    1 - (
      sum(rate(http_requests_total{job="payment-service", status!~"5.."}[1h]))
      /
      sum(rate(http_requests_total{job="payment-service"}[1h]))
    )
  ) / (1 - 0.999) > 14.4
)
```

---

## Multi-Window Multi-Burn-Rate Alerting

### Why Multiple Windows Are Needed

A single alert window cannot catch both fast and slow error budget burns effectively:

```
Single Window Problems
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  SHORT WINDOW ONLY (e.g., 5m):
  ✅ Catches fast burns quickly
  ❌ False positive on 5-minute spike (resolved before you respond)
  ❌ Misses 2% error rate over 7 days (slowly exhausting budget)

  LONG WINDOW ONLY (e.g., 6h):
  ✅ More stable, fewer false positives  
  ❌ 6 hours of 30% errors before alert fires → budget already gone
  ❌ Too slow for catastrophic failures

  MULTI-WINDOW (short AND long must both be elevated):
  ✅ Short window ensures fast detection
  ✅ Long window confirms sustained (not transient) burn
  ✅ Reduces false positives significantly
```

### Alert-Window-Burn Rate Table

| Alert Name | Short Window | Long Window | Burn Rate | Exhaustion Time | Severity |
|------------|-------------|------------|-----------|----------------|----------|
| `SLOBurnRateCritical` | 5m | 1h | 14.4× | ~2 hours | critical |
| `SLOBurnRateHigh` | 30m | 6h | 6× | ~5 hours | critical |
| `SLOBurnRateWarning` | 2h | 24h | 3× | ~10 days | warning |
| `SLOBurnRateInfo` | 6h | 3d | 1× | ~30 days | info |

### Complete Prometheus Alerting Rules YAML

```yaml
groups:
  - name: slo.burn_rate
    rules:
      # ─────────────────────────────────────────────────────────────────────
      # CRITICAL TIER 1: 14.4x burn rate
      # Short window: 5m | Long window: 1h
      # Significance: budget exhausted in ~2 hours
      # Action: Page on-call immediately
      # ─────────────────────────────────────────────────────────────────────
      - alert: SLOBurnRateCritical
        expr: |
          (
            1 - (
              sum(rate(http_requests_total{status!~"5.."}[5m])) by (job)
              / sum(rate(http_requests_total[5m])) by (job)
            )
          ) / (1 - 0.999) > 14.4
          AND
          (
            1 - (
              sum(rate(http_requests_total{status!~"5.."}[1h])) by (job)
              / sum(rate(http_requests_total[1h])) by (job)
            )
          ) / (1 - 0.999) > 14.4
        for: 2m
        labels:
          severity: critical
          slo_tier: "1"
          slo_target: "99.9"
        annotations:
          summary: "🔴 CRITICAL: SLO burn rate 14.4× on {{ $labels.job }}"
          description: >
            {{ $labels.job }} is burning through its error budget at 14.4× the
            sustainable rate. At this pace, the entire 30-day error budget will be
            exhausted in approximately 2 hours. Current 1-hour error rate exceeds
            the threshold for SLO {{ $labels.slo_target }}%.
            IMMEDIATE ACTION REQUIRED.
          runbook_url: "https://wiki.example.com/runbooks/slo-burn-rate"
          dashboard_url: "https://grafana.example.com/d/slo/slo-dashboard?var-job={{ $labels.job }}"

      # ─────────────────────────────────────────────────────────────────────
      # CRITICAL TIER 2: 6x burn rate
      # Short window: 30m | Long window: 6h
      # Significance: budget exhausted in ~5 hours
      # Action: Page on-call; start investigation
      # ─────────────────────────────────────────────────────────────────────
      - alert: SLOBurnRateHigh
        expr: |
          (
            1 - (
              sum(rate(http_requests_total{status!~"5.."}[30m])) by (job)
              / sum(rate(http_requests_total[30m])) by (job)
            )
          ) / (1 - 0.999) > 6
          AND
          (
            1 - (
              sum(rate(http_requests_total{status!~"5.."}[6h])) by (job)
              / sum(rate(http_requests_total[6h])) by (job)
            )
          ) / (1 - 0.999) > 6
        for: 15m
        labels:
          severity: critical
          slo_tier: "2"
          slo_target: "99.9"
        annotations:
          summary: "🔴 HIGH: SLO burn rate 6× on {{ $labels.job }}"
          description: >
            {{ $labels.job }} is burning error budget at 6× the sustainable rate.
            Budget will be exhausted in approximately 5 hours if the current rate
            continues. The 6-hour and 30-minute windows both confirm this elevated rate.
          runbook_url: "https://wiki.example.com/runbooks/slo-burn-rate"

      # ─────────────────────────────────────────────────────────────────────
      # WARNING: 3x burn rate
      # Short window: 2h | Long window: 24h
      # Significance: budget exhausted in ~10 days
      # Action: Investigate during business hours; do not page
      # ─────────────────────────────────────────────────────────────────────
      - alert: SLOBurnRateWarning
        expr: |
          (
            1 - (
              sum(rate(http_requests_total{status!~"5.."}[2h])) by (job)
              / sum(rate(http_requests_total[2h])) by (job)
            )
          ) / (1 - 0.999) > 3
          AND
          (
            1 - (
              sum(rate(http_requests_total{status!~"5.."}[24h])) by (job)
              / sum(rate(http_requests_total[24h])) by (job)
            )
          ) / (1 - 0.999) > 3
        for: 1h
        labels:
          severity: warning
          slo_tier: "3"
          slo_target: "99.9"
        annotations:
          summary: "🟡 WARNING: SLO burn rate 3× on {{ $labels.job }}"
          description: >
            {{ $labels.job }} error budget burn rate is 3× the sustainable level.
            The 30-day budget will be exhausted in approximately 10 days.
            Investigate during business hours and consider reducing deployment
            frequency until the error source is identified.
          runbook_url: "https://wiki.example.com/runbooks/slo-burn-rate"

      # ─────────────────────────────────────────────────────────────────────
      # INFO: 1x burn rate (budget on track to exhaust)
      # Long window: 3d
      # Significance: current rate will exhaust budget by month-end
      # Action: Create a ticket for next sprint
      # ─────────────────────────────────────────────────────────────────────
      - alert: SLOBurnRateInfo
        expr: |
          (
            1 - (
              sum(rate(http_requests_total{status!~"5.."}[3d])) by (job)
              / sum(rate(http_requests_total[3d])) by (job)
            )
          ) / (1 - 0.999) > 1
        for: 6h
        labels:
          severity: info
          slo_tier: "4"
          slo_target: "99.9"
        annotations:
          summary: "🔵 INFO: SLO budget burning at sustainable limit on {{ $labels.job }}"
          description: >
            {{ $labels.job }} is consuming error budget at or above the sustainable rate.
            At this pace, the full 30-day budget will be used by the end of the month.
            Create a reliability improvement ticket if this persists.
          runbook_url: "https://wiki.example.com/runbooks/slo-burn-rate"
```

---

## Error Budget Burn Rate

### Formula and Interpretation

```
Burn Rate Formula:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Burn Rate = current_error_rate / (1 - SLO_target)

  Example (SLO = 99.9%, error_budget = 0.001):

  Current error rate = 1.44%  (0.0144)
  Burn Rate = 0.0144 / 0.001 = 14.4

  Interpretation: consuming budget 14.4× faster than sustainable.
  Time to exhaustion = 30 days / 14.4 = ~2 hours

  current error rate = 0.2% (0.002)
  Burn Rate = 0.002 / 0.001 = 2
  Time to exhaustion = 30 days / 2 = 15 days
```

### Burn Rate Action Table

| Burn Rate | Time to Exhaustion (30d budget) | Error Rate (99.9% SLO) | Action Required |
|-----------|--------------------------------|------------------------|----------------|
| 0.1× | Never (budget accumulates) | 0.01% | No action |
| 1× | 30 days (exactly) | 0.1% | Monitor; create ticket if sustained |
| 2× | 15 days | 0.2% | Ticket; plan reliability work |
| 3× | 10 days | 0.3% | Warning alert; investigate during business hours |
| 6× | 5 days | 0.6% | Critical alert; investigate now |
| 14.4× | 2 hours | 1.44% | Critical alert; immediate page |
| 36× | 48 minutes | 3.6% | Likely an incident; all-hands |
| 100× | 17 minutes | 10% | Major incident; incident commander activated |

---

## SLA Negotiation Considerations

### Internal SLAs vs Customer-Facing SLAs

| Type | Audience | Consequence | Typical Tightness |
|------|---------|-------------|-------------------|
| Internal SLA | Other engineering teams | Incident review, priority reprioritization | Tighter than external |
| Customer-facing SLA | Paying customers | Financial credits, contract penalties | Looser than internal SLO |
| Partner/API SLA | Third-party integrators | Partner support, API credits | Varies |
| Vendor SLA | From your vendors (cloud, SaaS) | Credits against your vendor bill | Foundation for your own SLAs |

### Dependency SLAs

Your outgoing SLA cannot be better than the product of your dependencies' SLAs:

```
SLA Calculation with Dependencies
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Your dependencies:
  AWS EC2 SLA:      99.99%
  AWS RDS SLA:      99.95%
  Stripe API SLA:   99.99%
  Twilio SMS SLA:   99.95%

  Maximum theoretical availability:
  P(all available) = 0.9999 × 0.9995 × 0.9999 × 0.9995 = 99.88%

  Safe SLA you can offer customers: 99.5% (significant buffer)
  Your internal SLO: 99.9%
  SLA you'd breach: anything > 99.88%

  Rule: Your SLA target < Product of all dependency SLAs
```

### Consequences and Credits Table

| Availability (monthly) | Typical Credit | Notes |
|-----------------------|----------------|-------|
| ≥ SLA Target | 0% | Service performing as contracted |
| 99.0% – 99.5% (if SLA = 99.5%) | 5–10% | Minor breach |
| 95.0% – 99.0% | 10–25% | Significant downtime |
| 90.0% – 95.0% | 25–50% | Major outage affecting business |
| < 90.0% | 50–100% | Catastrophic; may trigger termination clause |

### How to Structure SLA Penalties

```
SLA Penalty Structure Best Practices
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  1. USE SERVICE CREDITS, NOT CASH REFUNDS
     → Credits keep the customer on your platform
     → Cash refunds are more complex and invite abuse

  2. CAP TOTAL CREDITS
     → Monthly SLA credits typically capped at 10–30% of monthly fee
     → Prevents gaming; you can't credit more than you charge

  3. REQUIRE CUSTOMER TO REQUEST CREDITS
     → Reduces administrative burden
     → Typical window: 30 days after incident

  4. EXCLUDE FORCE MAJEURE
     → Third-party outages (AWS, Cloudflare)
     → DDoS attacks beyond mitigation capacity
     → Scheduled maintenance windows (notify 72h in advance)

  5. MEASUREMENT METHOD
     → Define exactly HOW availability is measured
     → Prometheus? Synthetic monitors? Third-party uptime monitor?
     → Dispute resolution: who is the authoritative data source?
```

---

## Example SLOs

### Web API

```yaml
# slo-definitions.yaml — Web API
slos:
  - name: "payment-api-availability"
    service: "payment-service"
    description: "Fraction of HTTP requests that return non-5xx responses"
    sli:
      type: availability
      metric: http_requests_total
      good_selector: 'status!~"5.."'
      total_selector: ""
    target: 0.999           # 99.9% over rolling 30-day window
    error_budget_minutes: 43.8

  - name: "payment-api-latency-p50"
    service: "payment-service"
    description: "50th percentile of requests completing under 200ms"
    sli:
      type: latency
      metric: http_request_duration_seconds
      threshold: 0.2        # 200ms
      percentile: 50
    target: 0.95            # 95% of requests under 200ms

  - name: "payment-api-latency-p99"
    service: "payment-service"
    description: "99th percentile of requests completing under 2s"
    sli:
      type: latency
      metric: http_request_duration_seconds
      threshold: 2.0        # 2 seconds
      percentile: 99
    target: 0.99            # 99% of requests under 2s

  - name: "payment-api-error-rate"
    service: "payment-service"
    description: "Fraction of requests that are not errors (inverted error rate)"
    sli:
      type: error_rate
      metric: http_requests_total
      error_selector: 'status=~"5.."'
    target: 0.999           # Error rate < 0.1%
```

### Batch Job

```yaml
slos:
  - name: "invoice-job-completion-time"
    service: "invoice-generator"
    description: "Monthly invoice generation job completes within 4 hours of trigger"
    sli:
      type: throughput
      metric: batch_job_duration_seconds
      threshold: 14400      # 4 hours in seconds
    target: 0.99            # 99% of job runs complete within 4h

  - name: "invoice-job-success-rate"
    service: "invoice-generator"
    description: "Fraction of invoice generation jobs that complete without error"
    sli:
      type: availability
      metric: batch_job_total
      good_selector: 'status="success"'
    target: 0.999           # 99.9% success rate

  - name: "invoice-job-freshness"
    service: "invoice-generator"
    description: "Invoices are available within 6 hours of billing period close"
    sli:
      type: freshness
      threshold_hours: 6
    target: 0.99
```

### Data Pipeline

```yaml
slos:
  - name: "event-pipeline-throughput"
    service: "analytics-pipeline"
    description: "Events processed per second meets minimum throughput"
    sli:
      type: throughput
      minimum_events_per_second: 10000
    target: 0.99            # 99% of time throughput > 10K events/s

  - name: "event-pipeline-freshness"
    service: "analytics-pipeline"
    description: "Latest data in the warehouse is no more than 1 hour old"
    sli:
      type: freshness
      threshold_minutes: 60
    target: 0.99            # 99% of time, data < 1 hour old

  - name: "event-pipeline-correctness"
    service: "analytics-pipeline"
    description: "Events pass schema validation without errors"
    sli:
      type: correctness
      metric: pipeline_validation_total
      good_selector: 'status="valid"'
    target: 0.9999          # 99.99% of events pass validation
```

### Streaming Service

```yaml
slos:
  - name: "streaming-availability"
    service: "video-streaming"
    description: "Stream starts succeed without error"
    sli:
      type: availability
      metric: stream_start_total
      good_selector: 'result="success"'
    target: 0.999

  - name: "streaming-startup-latency"
    service: "video-streaming"
    description: "Time to first frame under 3 seconds"
    sli:
      type: latency
      threshold: 3.0
    target: 0.95            # 95% of streams start within 3s

  - name: "streaming-data-loss"
    service: "video-streaming"
    description: "Zero dropped frames per stream session"
    sli:
      type: correctness
      metric: stream_frames_total
      good_selector: 'status="delivered"'
    target: 0.9999          # 99.99% of frames delivered

  - name: "streaming-rebuffering"
    service: "video-streaming"
    description: "Rebuffering ratio: rebuffering time / total playback time < 0.5%"
    sli:
      type: custom
      expr: |
        1 - (
          sum(rate(stream_rebuffering_seconds_total[5m]))
          /
          sum(rate(stream_playback_seconds_total[5m]))
        )
    target: 0.995           # Rebuffering ratio < 0.5%
```

---

## Google SRE Approach to SLOs

### Key Principles from the SRE Book

Google's Site Reliability Engineering book (the definitive reference for SRE practice) establishes these core SLO principles:

| Principle | Explanation |
|-----------|-------------|
| **Start with user happiness** | SLOs should measure whether users are satisfied with the service, not whether the infrastructure is healthy |
| **Fewer is better** | 1–3 SLOs per critical user journey. Too many SLOs dilute focus and create cognitive overhead |
| **Don't over-achieve** | If the service performs better than the SLO, use the surplus budget for innovation, not to tighten the SLO |
| **Iteration over perfection** | Your first SLOs will be wrong. Commit to a review cycle. Evolve them. |
| **Error budget as neutral arbiter** | The error budget, not opinions or hierarchy, decides when to ship features vs. invest in reliability |

### The "Toil" Concept and SLOs

**Toil** is manual, repetitive, automatable work that grows linearly with service scale. SLOs provide a forcing function against toil:

```
Toil and SLOs relationship:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  High toil = manual restarts, manual rollbacks, manual config changes
           → Each incident consumes error budget
           → Error budget exhaustion → freeze features
           → Engineers must automate toil to regain ability to ship
           → SLOs create financial (velocity) incentive to reduce toil

  SRE Rule: Toil should be < 50% of an SRE's time.
  SLO mechanism enforces this indirectly:
  Too much toil → budget burns → feature freeze → automation prioritized.
```

### Error Budget Policy in Practice

```
Google's Error Budget Policy Process
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Budget is HEALTHY (>50% remaining):
  → No restrictions on deployments
  → Experiments and risky changes allowed
  → Focus on feature velocity

  Budget is DEPLETED (0% remaining):
  → Product team and SRE team BOTH agreed in advance on this policy
  → Feature releases halted automatically (not by executive decree)
  → All engineering effort redirected to:
     1. Postmortems and root cause analysis
     2. Reducing toil (automation)
     3. Improving testing and deployment safety
  → Releases resume only when:
     a. SLO is met again over the defined compliance window
     b. Root cause is understood and fixed

  KEY INSIGHT: Policy agreed BEFORE crisis removes blame/politics.
  "The error budget says we stop. It's not personal."
```

### The SLO Review Cycle

```
Quarterly SLO Review Checklist
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  1. DATA REVIEW
     □ Did we breach the SLO in the past quarter?
     □ How many times was the error budget >50% consumed?
     □ Were the breaches from expected or unexpected causes?

  2. SLO TARGET REVIEW
     □ Is the SLO target still appropriate for user expectations?
     □ Has the business changed (new markets, premium customers)?
     □ Is the SLO too easy? (Never close to breaching = too loose)
     □ Is the SLO too hard? (Frequently breaching = too tight or under-invested)

  3. SLI REVIEW
     □ Is the SLI measurement still accurate?
     □ Are we excluding the right traffic (synthetics, health checks)?
     □ Should we add new SLIs for new features?

  4. ERROR BUDGET POLICY REVIEW
     □ Did the policy activate? Was it followed?
     □ Were the thresholds (50%, 25%, 10%) appropriate?
     □ Does the policy need updates?

  5. ROADMAP INPUT
     □ What reliability investments are needed next quarter?
     □ What is the reliability/feature ratio for the next sprint?
```

---

## Next Steps

1. **Define your first SLO** — Pick one critical user journey. Measure the SLI for 30 days. Set the target 10–20% below your worst month. Document the error budget policy.
2. **Implement recording rules** — Deploy the Prometheus recording rules from this guide to compute rolling SLI values efficiently.
3. **Build the SLO dashboard** — Create a Grafana dashboard with: SLI value (30d rolling), error budget remaining gauge, burn rate panels.
4. **Configure multi-window burn rate alerts** — Deploy all four alert tiers from the alerting section. Integrate with your Alertmanager routing.
5. **Write the error budget policy** — Document in a team wiki what happens at 50%, 25%, and 0% remaining. Get sign-off from product and engineering leadership.
6. **Schedule quarterly SLO reviews** — Put the SLO review on the calendar for the end of each quarter. Review the checklist above.
7. **Expand to all critical services** — Start with 1–2 services and the availability SLI. Expand to latency SLIs and additional services over time.

**Related guides in this series:**
- [01-METRICS.md](01-METRICS.md) — Prometheus and PromQL foundations for SLI calculations
- [05-ALERTING.md](05-ALERTING.md) — Multi-window burn rate alerting implementation
- [06-APM.md](06-APM.md) — APM tools for measuring latency and error SLIs

---

## Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0.0 | 2025-01-01 | Platform Engineering | Initial document |
| 1.1.0 | 2025-01-25 | Platform Engineering | Added complete PromQL SLI examples section |
| 1.2.0 | 2025-02-20 | Platform Engineering | Added multi-window burn rate alert YAML |
| 1.3.0 | 2025-03-15 | Platform Engineering | Added example SLO definitions (web API, batch, pipeline, streaming) |
| 1.4.0 | 2025-04-10 | Platform Engineering | Expanded SLA negotiation section with dependency SLA calculation |
| 1.5.0 | 2025-05-01 | Platform Engineering | Added Google SRE principles and SLO review cycle checklist |
| 1.6.0 | 2025-06-01 | Platform Engineering | Added Grafana panel JSON configuration examples |
