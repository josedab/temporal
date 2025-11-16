# RFC-0004: Metrics Dashboard

**Status:** Draft | **Effort:** 3-5 days | **Impact:** Medium
**Author:** Analysis Team | **Created:** 2025-11-16

## Summary

Create comprehensive operator dashboard for monitoring Temporal cluster health, performance, and capacity.

## Motivation

Operators lack visibility into cluster health. Manual metric correlation is time-consuming. Need centralized dashboard showing: workflow throughput, task latency, queue depths, database performance, shard distribution.

## Proposed Solution

**Grafana Dashboard with:**
- Workflow metrics: start/complete rates, latency, errors
- Task metrics: dispatch latency, sync match rate
- Queue metrics: depth, processing lag
- Database: connection pool, query latency
- Alerts: queue lag, database slow queries, error rate spikes
**Tech:** Grafana + Prometheus + pre-built dashboards

## Implementation Plan

**Day 1-2:** Design dashboard layout, metrics selection
**Day 3-4:** Implement Grafana dashboards, test with real data
**Day 5:** Documentation, rollout to staging
**Total:** 1 week

## Success Metrics

- ✅ Dashboard deployed to all clusters
- ✅ Alerts configured and tested
- ✅ MTTR (Mean Time to Remediation) reduced by 50%
- ✅ Operator satisfaction 4+/5

## Risks

**Low:** Dashboard creation, no code changes. Risk: dashboard doesn't cover all use cases. Mitigation: iterative feedback.

---

**Next:** See [Prioritization Matrix](00-prioritization-matrix.md)
