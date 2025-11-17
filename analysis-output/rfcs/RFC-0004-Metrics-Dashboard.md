# RFC-0004: Operator Metrics Dashboard

**Status:** Draft
**Author:** Analysis Team
**Created:** 2025-11-16
**Commit SHA:** `0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3`

---

## Summary

Create comprehensive Grafana dashboards for monitoring Temporal cluster health, performance, and capacity, providing operators with actionable visibility into system behavior.

---

## Motivation

### Current Problem

Operators face significant challenges when monitoring Temporal clusters:

**1. Metric Fragmentation**
- Metrics scattered across Prometheus, application logs, database monitoring
- No single pane of glass for cluster health
- Difficult to correlate workflow issues with underlying infrastructure

**2. Manual Correlation is Time-Consuming**
```
Scenario: "Workflows are slow"

Current Investigation Process:
1. Check Prometheus for workflow_start_latency → High
2. Check history service metrics → Normal
3. Check matching service metrics → High queue depth!
4. Check database metrics → Connection pool saturated
5. Cross-reference timestamps across 4 different tools
6. Identify root cause after 30+ minutes

Total Time: 30-60 minutes per incident
```

**3. Lack of Alerting Context**
- Alerts fire without context (e.g., "High latency" - where? why?)
- No visual correlation between symptoms and causes
- On-call engineers start from scratch each incident

**4. Capacity Planning Blindness**
- No visibility into resource utilization trends
- Can't predict when cluster will hit capacity
- Reactive scaling instead of proactive

### Impact

- **High MTTR:** Mean Time to Remediation = 45-60 minutes
- **Inefficient Operations:** 20% of operator time spent correlating metrics
- **Incident Escalation:** Issues escalate unnecessarily due to lack of visibility
- **Poor Capacity Planning:** Unexpected capacity issues, emergency scaling

### User Story

```
As a Temporal operator,
I want a single dashboard showing cluster health,
So that I can quickly identify and resolve issues
Without manually correlating metrics across multiple tools.
```

---

## Detailed Design

### Dashboard Architecture

```
┌─────────────────────────────────────────────────────┐
│           Temporal Operator Dashboard               │
├─────────────────────────────────────────────────────┤
│  Overview          │  Details (drill-down)          │
│  ────────────────  │  ──────────────────────────    │
│  - Cluster Health  │  - Service Metrics             │
│  - Throughput      │  - Shard Details               │
│  - Latency         │  - Queue Analysis              │
│  - Error Rate      │  - Database Performance        │
│                    │  - Worker Metrics              │
└─────────────────────────────────────────────────────┘
         │
         ▼
  ┌──────────────┐
  │  Prometheus  │  (Data source)
  └──────────────┘
```

### Dashboard 1: Cluster Overview (Primary Dashboard)

**Purpose:** At-a-glance cluster health and performance

#### Panel 1: Cluster Health Status

```
┌─────────────────────────────────────────┐
│  Cluster Health                         │
│  ───────────────────────────────────    │
│  Overall Status:  🟢 Healthy            │
│                                         │
│  Frontend:        🟢 3/3 healthy        │
│  History:         🟢 4/4 healthy        │
│  Matching:        🟢 2/2 healthy        │
│  Worker:          🟢 1/1 healthy        │
│                                         │
│  Database:        🟡 95% conn pool      │
│  Elasticsearch:   🟢 Healthy            │
└─────────────────────────────────────────┘

Metrics Used:
- up{job="temporal-frontend"}
- up{job="temporal-history"}
- up{job="temporal-matching"}
- up{job="temporal-worker"}
```

**Panel Configuration:**
```json
{
  "title": "Cluster Health",
  "type": "stat",
  "targets": [
    {
      "expr": "count(up{job=~\"temporal-.*\"} == 1) / count(up{job=~\"temporal-.*\"})",
      "legendFormat": "Health %"
    }
  ],
  "thresholds": {
    "mode": "absolute",
    "steps": [
      {"value": 0, "color": "red"},
      {"value": 0.8, "color": "yellow"},
      {"value": 0.95, "color": "green"}
    ]
  }
}
```

#### Panel 2: Workflow Throughput

```
┌─────────────────────────────────────────┐
│  Workflow Throughput (ops/sec)          │
│  ───────────────────────────────────    │
│  1.2K ▲ ──────────────────────          │
│       │        ╱╲                        │
│  800  │       ╱  ╲    ╱╲                 │
│       │   ╱──╯    ╲──╯  ╲                │
│  400  │  ╱                ╲╱╲            │
│       │─╯                     ╲──        │
│    0  └──────────────────────────        │
│       00:00  06:00  12:00  18:00        │
│                                         │
│  Current: 1,234 ops/s                   │
│  Peak:    1,456 ops/s (14:23)           │
│  Average:   987 ops/s                   │
└─────────────────────────────────────────┘

Metrics:
- rate(temporal_workflow_start_total[1m])
- rate(temporal_workflow_complete_total[1m])
```

**Query:**
```promql
# Workflow starts per second
sum(rate(temporal_workflow_start_total[1m])) by (namespace)

# Workflow completions per second
sum(rate(temporal_workflow_complete_total[1m])) by (namespace)
```

#### Panel 3: Task Dispatch Latency

```
┌─────────────────────────────────────────┐
│  Task Dispatch Latency (P50/P99)        │
│  ───────────────────────────────────    │
│  100ms│                                 │
│       │              P99 ────────       │
│   50ms│         ╱────                   │
│       │    ╱───╯                        │
│   10ms│───╯ P50 ──────────────          │
│       │                                 │
│    0ms└──────────────────────────       │
│       00:00  06:00  12:00  18:00       │
│                                         │
│  Sync Match Rate: 87%  🟢               │
└─────────────────────────────────────────┘

Metrics:
- histogram_quantile(0.50, temporal_task_dispatch_latency_bucket)
- histogram_quantile(0.99, temporal_task_dispatch_latency_bucket)
- temporal_matching_sync_match_rate
```

#### Panel 4: Error Rate

```
┌─────────────────────────────────────────┐
│  Error Rate (errors/min)                │
│  ───────────────────────────────────    │
│   10 │                                  │
│      │           🔴                     │
│    5 │      ╱────╲                      │
│      │  ╱──╯      ╲                     │
│    1 │─╯           ╲────────            │
│      │                                  │
│    0 └──────────────────────────        │
│      00:00  06:00  12:00  18:00        │
│                                         │
│  Spike Detected: 12:15 (+450%)          │
│  Type: ActivityTaskFailed               │
└─────────────────────────────────────────┘

Metrics:
- rate(temporal_workflow_failed_total[1m])
- rate(temporal_activity_failed_total[1m])
```

#### Panel 5: Queue Depths

```
┌─────────────────────────────────────────┐
│  Queue Depths (tasks pending)           │
│  ───────────────────────────────────    │
│  Transfer Queue:   1,234  🟢            │
│  Timer Queue:        456  🟢            │
│  Visibility Queue:    23  🟢            │
│  Replication Queue:    0  🟢            │
│                                         │
│  Max Capacity:    10,000                │
│  Utilization:        17%                │
└─────────────────────────────────────────┘

Metrics:
- temporal_transfer_queue_depth
- temporal_timer_queue_depth
- temporal_visibility_queue_depth
- temporal_replication_queue_depth
```

---

### Dashboard 2: Service Deep Dive

#### Frontend Service Metrics

```json
{
  "dashboard": "Frontend Service Details",
  "panels": [
    {
      "title": "Request Rate by API",
      "query": "sum(rate(temporal_frontend_request_total[1m])) by (operation)",
      "type": "graph"
    },
    {
      "title": "API Latency (P99)",
      "query": "histogram_quantile(0.99, temporal_frontend_latency_bucket) by (operation)",
      "type": "graph"
    },
    {
      "title": "Rate Limiting Events",
      "query": "sum(rate(temporal_frontend_rate_limit_exceeded_total[1m])) by (namespace)",
      "type": "graph"
    },
    {
      "title": "gRPC Connection Count",
      "query": "temporal_frontend_grpc_connections",
      "type": "stat"
    }
  ]
}
```

#### History Service Metrics

```json
{
  "dashboard": "History Service Details",
  "panels": [
    {
      "title": "Shard Distribution",
      "query": "count(temporal_history_shard_ownership) by (host)",
      "type": "pie"
    },
    {
      "title": "Mutable State Cache Hit Rate",
      "query": "rate(temporal_history_cache_hit_total[1m]) / rate(temporal_history_cache_requests_total[1m])",
      "type": "gauge"
    },
    {
      "title": "Queue Processing Lag",
      "query": "temporal_history_queue_lag_seconds",
      "type": "graph",
      "alert": {
        "condition": "> 300s",
        "severity": "warning"
      }
    },
    {
      "title": "Workflow Execution Cache Size",
      "query": "temporal_history_workflow_cache_size",
      "type": "stat"
    }
  ]
}
```

#### Matching Service Metrics

```json
{
  "dashboard": "Matching Service Details",
  "panels": [
    {
      "title": "Task Queue Backlog",
      "query": "temporal_matching_task_queue_backlog by (task_queue)",
      "type": "table"
    },
    {
      "title": "Sync vs Async Match Ratio",
      "query": "sum(rate(temporal_matching_sync_match_total[1m])) / sum(rate(temporal_matching_task_dispatched_total[1m]))",
      "type": "gauge",
      "thresholds": {
        "good": "> 0.8",
        "warn": "0.5-0.8",
        "critical": "< 0.5"
      }
    },
    {
      "title": "Poller Count by Task Queue",
      "query": "temporal_matching_pollers_total by (task_queue)",
      "type": "graph"
    }
  ]
}
```

---

### Dashboard 3: Database Performance

```json
{
  "dashboard": "Database Performance",
  "panels": [
    {
      "title": "Database Query Latency",
      "targets": [
        {
          "expr": "histogram_quantile(0.99, temporal_persistence_latency_bucket{operation=\"UpdateWorkflowExecution\"})",
          "legendFormat": "Update Workflow (P99)"
        },
        {
          "expr": "histogram_quantile(0.99, temporal_persistence_latency_bucket{operation=\"AppendHistoryEvents\"})",
          "legendFormat": "Append Events (P99)"
        }
      ],
      "type": "graph",
      "yAxis": {"format": "ms"}
    },
    {
      "title": "Connection Pool Utilization",
      "query": "temporal_persistence_active_connections / temporal_persistence_max_connections",
      "type": "gauge",
      "thresholds": {
        "steps": [
          {"value": 0, "color": "green"},
          {"value": 0.7, "color": "yellow"},
          {"value": 0.9, "color": "red"}
        ]
      }
    },
    {
      "title": "Database Operations/sec",
      "query": "sum(rate(temporal_persistence_requests_total[1m])) by (operation)",
      "type": "graph"
    },
    {
      "title": "Database Error Rate",
      "query": "sum(rate(temporal_persistence_errors_total[1m])) by (error_type)",
      "type": "graph",
      "alert": {
        "condition": "> 10",
        "message": "High database error rate detected"
      }
    }
  ]
}
```

---

## Alert Rules

### Critical Alerts (Page Immediately)

**Alert 1: Cluster Health Below 80%**
```yaml
- alert: TemporalClusterUnhealthy
  expr: |
    count(up{job=~"temporal-.*"} == 1) / count(up{job=~"temporal-.*"}) < 0.8
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "Temporal cluster health below 80%"
    description: "Only {{ $value | humanizePercentage }} of Temporal services are healthy"
```

**Alert 2: Queue Processing Lag**
```yaml
- alert: TemporalQueueLagHigh
  expr: temporal_history_queue_lag_seconds > 300
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Queue processing lag exceeds 5 minutes"
    description: "{{ $labels.queue_type }} queue lag: {{ $value }}s"
```

**Alert 3: Database Connection Pool Exhausted**
```yaml
- alert: DatabaseConnectionPoolExhausted
  expr: |
    temporal_persistence_active_connections / temporal_persistence_max_connections > 0.95
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "Database connection pool near exhaustion"
    description: "Connection pool utilization: {{ $value | humanizePercentage }}"
```

### Warning Alerts (Investigate Soon)

**Alert 4: High Task Dispatch Latency**
```yaml
- alert: HighTaskDispatchLatency
  expr: |
    histogram_quantile(0.99, temporal_task_dispatch_latency_bucket) > 500
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "Task dispatch P99 latency > 500ms"
    description: "Current P99 latency: {{ $value }}ms"
```

**Alert 5: Low Sync Match Rate**
```yaml
- alert: LowSyncMatchRate
  expr: |
    sum(rate(temporal_matching_sync_match_total[5m]))
    /
    sum(rate(temporal_matching_task_dispatched_total[5m])) < 0.5
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "Sync match rate below 50%"
    description: "Current sync match rate: {{ $value | humanizePercentage }}"
```

**Alert 6: Workflow Error Rate Spike**
```yaml
- alert: WorkflowErrorRateSpike
  expr: |
    (rate(temporal_workflow_failed_total[5m])
    /
    rate(temporal_workflow_complete_total[5m])) > 0.1
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Workflow error rate > 10%"
    description: "Error rate: {{ $value | humanizePercentage }}"
```

---

## Implementation Plan

### Phase 1: Dashboard Creation (Day 1-2)

**Day 1: Design & Setup**
1. Install Grafana (if not present)
   ```bash
   docker run -d -p 3000:3000 grafana/grafana
   ```

2. Configure Prometheus data source
   ```json
   {
     "name": "Temporal Prometheus",
     "type": "prometheus",
     "url": "http://prometheus:9090",
     "access": "proxy"
   }
   ```

3. Create dashboard templates
   - Cluster Overview (primary)
   - Service Details (drill-down)
   - Database Performance
   - Capacity Planning

**Day 2: Panel Implementation**
1. Implement all panels defined above
2. Configure time ranges, refresh rates
3. Set up variables for filtering:
   ```json
   {
     "variables": [
       {
         "name": "namespace",
         "type": "query",
         "query": "label_values(temporal_workflow_start_total, namespace)"
       },
       {
         "name": "cluster",
         "type": "query",
         "query": "label_values(up{job=~\"temporal-.*\"}, cluster)"
       }
     ]
   }
   ```

### Phase 2: Alert Configuration (Day 3)

1. Create alert rules in Prometheus
   ```yaml
   # /etc/prometheus/rules/temporal.yml
   groups:
     - name: temporal
       interval: 30s
       rules:
         - alert: TemporalClusterUnhealthy
           # ... (see alert definitions above)
   ```

2. Configure alert routing
   ```yaml
   # /etc/alertmanager/config.yml
   route:
     group_by: ['alertname', 'cluster']
     group_wait: 30s
     group_interval: 5m
     repeat_interval: 12h
     receiver: 'temporal-oncall'

   receivers:
     - name: 'temporal-oncall'
       pagerduty_configs:
         - service_key: '<YOUR_KEY>'
           description: '{{ .GroupLabels.alertname }}: {{ .Annotations.summary }}'
   ```

3. Test alerting
   ```bash
   # Trigger test alert
   curl -X POST http://prometheus:9090/api/v1/alerts \
     -d '{"alerts":[{"labels":{"alertname":"TestAlert","severity":"warning"}}]}'
   ```

### Phase 3: Testing & Validation (Day 4)

1. Load testing
   ```bash
   # Generate load to verify dashboard accuracy
   temporal workflow start \
     --task-queue load-test \
     --workflow-type LoadTestWorkflow \
     --workflow-id-reuse-policy AllowDuplicate \
     --count 1000
   ```

2. Verify metrics accuracy
   - Compare dashboard metrics with raw Prometheus queries
   - Validate alert thresholds trigger correctly
   - Test drill-down navigation

3. User acceptance testing
   - Operators review dashboards
   - Collect feedback on missing metrics
   - Iterate on panel layouts

### Phase 4: Documentation & Rollout (Day 5)

1. Create operator documentation
   ```markdown
   # Temporal Operator Dashboard Guide

   ## Accessing Dashboards
   URL: https://grafana.company.com/d/temporal-overview

   ## Understanding Metrics
   - Green: Healthy (within normal parameters)
   - Yellow: Warning (investigate soon)
   - Red: Critical (immediate action required)

   ## Common Investigation Workflows

   ### Slow Workflows
   1. Check Task Dispatch Latency panel
   2. If high, drill into Matching Service dashboard
   3. Check poller count and queue backlog
   4. Investigate worker capacity

   ### High Error Rate
   1. Check Error Rate panel
   2. Note error type (workflow vs activity)
   3. Drill into Service Details for namespace breakdown
   4. Check logs for error details
   ```

2. Rollout to production
   ```bash
   # Export dashboard JSON
   curl -X GET http://grafana:3000/api/dashboards/uid/temporal-overview \
     -H "Authorization: Bearer $GRAFANA_API_KEY" \
     > temporal-dashboard.json

   # Import to production Grafana
   curl -X POST http://prod-grafana:3000/api/dashboards/db \
     -H "Authorization: Bearer $PROD_API_KEY" \
     -H "Content-Type: application/json" \
     -d @temporal-dashboard.json
   ```

3. Training session
   - 1-hour training for operators
   - Walkthrough of each dashboard
   - Practice using dashboards during simulated incidents

---

## Example Dashboard JSON

**Cluster Overview Dashboard:**
```json
{
  "dashboard": {
    "title": "Temporal Cluster Overview",
    "tags": ["temporal", "overview"],
    "timezone": "browser",
    "refresh": "30s",
    "time": {
      "from": "now-6h",
      "to": "now"
    },
    "panels": [
      {
        "id": 1,
        "title": "Cluster Health",
        "type": "stat",
        "gridPos": {"h": 4, "w": 6, "x": 0, "y": 0},
        "targets": [
          {
            "expr": "count(up{job=~\"temporal-.*\"} == 1) / count(up{job=~\"temporal-.*\"})",
            "refId": "A"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percentunit",
            "thresholds": {
              "mode": "absolute",
              "steps": [
                {"value": 0, "color": "red"},
                {"value": 0.8, "color": "yellow"},
                {"value": 0.95, "color": "green"}
              ]
            }
          }
        }
      },
      {
        "id": 2,
        "title": "Workflow Throughput",
        "type": "graph",
        "gridPos": {"h": 8, "w": 12, "x": 0, "y": 4},
        "targets": [
          {
            "expr": "sum(rate(temporal_workflow_start_total[1m]))",
            "legendFormat": "Starts/sec",
            "refId": "A"
          },
          {
            "expr": "sum(rate(temporal_workflow_complete_total[1m]))",
            "legendFormat": "Completions/sec",
            "refId": "B"
          }
        ],
        "yaxes": [
          {"format": "ops", "label": "Operations/sec"}
        ]
      }
      // ... more panels
    ]
  }
}
```

---

## Success Metrics

### Operational Metrics

| Metric | Baseline | Target | Measurement |
|--------|----------|--------|-------------|
| **MTTR** | 45-60 min | 15-20 min | Incident ticket timestamps |
| **Dashboard Adoption** | 0% | 100% | Operators using dashboard daily |
| **Alert Accuracy** | N/A | <5% false positives | Alert review |
| **Incident Escalation** | 30% | <10% | Percentage escalated to engineering |

### User Satisfaction

- **Survey:** Operator satisfaction survey (target: 4+/5)
- **Feedback:** Weekly feedback sessions for first month
- **Usage:** Dashboard views per day (target: 50+ views)

### Technical Metrics

- **Dashboard Load Time:** <2 seconds
- **Query Performance:** All queries <1 second
- **Alert Latency:** Alerts fire within 2 minutes of condition

---

## Backwards Compatibility

**No breaking changes:**
- Dashboard is additive (no code changes to Temporal server)
- Existing metrics unchanged
- Alert rules optional (operators can disable)

**Migration:** None required

---

## Alternatives Considered

### Alternative 1: Custom Web UI

**Approach:** Build custom React dashboard

**Pros:**
- Full control over UX
- Integrated with Temporal UI

**Cons:**
- Development time: 4-6 weeks vs. 1 week
- Maintenance burden
- Less ecosystem support

**Verdict:** Rejected - Grafana is industry standard, well-supported

### Alternative 2: Cloud Monitoring (DataDog, New Relic)

**Approach:** Use SaaS monitoring platform

**Pros:**
- Fully managed
- Advanced features (APM, log correlation)

**Cons:**
- Cost: $500-2000/month
- Vendor lock-in
- Data privacy concerns

**Verdict:** Rejected for initial implementation (can add later)

### Alternative 3: Prometheus + PromQL CLI

**Approach:** Operators query Prometheus directly

**Pros:**
- No additional infrastructure
- Maximum flexibility

**Cons:**
- Steep learning curve (PromQL)
- No visualization
- No alerting context

**Verdict:** Rejected - too operator-unfriendly

---

## Open Questions

1. **Custom metrics:** Do operators need custom metrics beyond standard set?
   - **Answer:** Start with standard, iterate based on feedback

2. **Multi-cluster:** How to handle multi-cluster deployments?
   - **Answer:** Use `cluster` variable, separate dashboards per cluster initially

3. **Historical data:** How long to retain dashboard data?
   - **Answer:** 30 days for high-resolution, 1 year for aggregated (Prometheus default)

4. **Access control:** Who should have access to dashboards?
   - **Answer:** All operators (read-only), engineering leads (edit)

---

## Risk Assessment

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|-----------|
| **Dashboard doesn't cover all use cases** | Medium | Medium | Iterative feedback, monthly updates |
| **Alert fatigue from false positives** | High | Medium | Conservative thresholds initially, tune based on data |
| **Grafana performance issues** | Medium | Low | Use fast queries, pre-aggregated metrics |
| **Operators don't adopt dashboard** | High | Low | Training, make it default home page |

---

## Rollback Strategy

**If dashboard causes issues:**

1. **Disable alerts** (immediate)
   ```bash
   # In Prometheus config
   alerting:
     alertmanagers: []  # Disable alerting
   ```

2. **Revert to previous Grafana state**
   ```bash
   # Restore dashboard backup
   curl -X POST http://grafana:3000/api/dashboards/db \
     -d @backup/previous-dashboard.json
   ```

3. **No server impact** - Dashboard is read-only, no risk to Temporal server

---

## References

- [Grafana Documentation](https://grafana.com/docs/)
- [Prometheus Alerting](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/)
- [Temporal Metrics](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/common/metrics/defs.go)
- [SRE Book - Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)

---

**Status:** Draft - Ready for review and approval
**Estimated Effort:** 1 week (1 engineer)
**Estimated Cost:** $10K (engineer time) + $0 (infrastructure, uses existing Grafana)
**Next Steps:**
1. Stakeholder review
2. Operator feedback on dashboard mockups
3. Implementation kick-off
