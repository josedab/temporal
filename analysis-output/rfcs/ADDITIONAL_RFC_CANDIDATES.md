# Additional High-Impact RFC Candidates

## Analysis of Coverage Gaps

After reviewing the existing 10 RFCs, here are significant high-impact areas not yet addressed:

---

## 🔴 Critical Priority

### RFC-0011: Cost Optimization and Storage Management
**Impact:** VERY HIGH | **Effort:** 6-8 weeks

**Problem:**
- History grows unbounded (terabytes for large deployments)
- Database costs can reach $50K-100K/month at scale
- No automated history compaction or archival
- Storage costs grow linearly with workflow count

**Proposed Solutions:**
- Automated history archival to S3/cold storage
- History compaction (remove redundant events)
- Configurable retention policies per namespace
- Storage tiering (hot/warm/cold)
- Cost attribution per namespace/workflow type

**Why Critical:** Directly impacts TCO. Enterprises cite storage costs as #1 concern.

---

### RFC-0012: Multi-Region Active-Active Architecture
**Impact:** VERY HIGH | **Effort:** 12-16 weeks

**Problem:**
- Single-region deployment = single point of failure
- No automated cross-region failover
- RTO/RPO requirements not met for tier-1 services
- Disaster recovery is manual and slow

**Proposed Solutions:**
- Active-active multi-region replication
- Global namespace routing
- Automated failover (RTO < 5 minutes)
- Conflict resolution for concurrent updates
- Cross-region workflow execution

**Why Critical:** Required for enterprise SLAs (99.99%+ uptime). Competitive disadvantage without it.

---

## 🟠 High Priority

### RFC-0013: Multi-Tenancy Resource Isolation & Fair Scheduling
**Impact:** HIGH | **Effort:** 8-10 weeks

**Problem:**
- No resource quotas per namespace
- One noisy tenant can impact all tenants
- No fair scheduling guarantees
- No cost attribution for billing

**Proposed Solutions:**
- Per-namespace CPU/memory quotas
- Fair scheduling algorithm (weighted fair queuing)
- Rate limiting per tenant
- Resource usage metering for billing
- Priority-based task scheduling

**Why High Priority:** Essential for SaaS providers building on Temporal. Current model doesn't support true multi-tenancy.

---

### RFC-0014: Workflow State Migration & Schema Evolution
**Impact:** HIGH | **Effort:** 6-8 weeks

**Problem:**
- Can't migrate long-running workflows to new code
- Breaking changes require workflow termination
- No schema evolution for workflow state
- Stuck with bugs in production workflows for months/years

**Proposed Solutions:**
- State migration framework
- Schema versioning for workflow state
- Migration strategies (transform, replay, side-by-side)
- Automated migration testing
- Rollback capabilities

**Why High Priority:** Critical for long-running workflows (months/years). Bugs or improvements stuck until workflow completes.

---

### RFC-0015: Advanced Observability & Debugging Tools
**Impact:** HIGH | **Effort:** 6-8 weeks

**Problem:**
- Debugging complex workflow failures is difficult
- No workflow execution replay in UI
- Limited tracing correlation
- No performance profiling per workflow
- Hard to diagnose "why did this workflow take 3 hours?"

**Proposed Solutions:**
- Time-travel debugging (replay workflow in debugger)
- Workflow execution flamegraphs
- Critical path analysis (what's blocking completion?)
- Enhanced distributed tracing integration
- Workflow-level cost attribution
- Visual workflow execution inspector

**Why High Priority:** Developer productivity. Hours spent debugging could be minutes with better tools.

---

## 🟡 Medium-High Priority

### RFC-0016: Query Performance at Scale (100M+ workflows)
**Impact:** MEDIUM-HIGH | **Effort:** 8-10 weeks

**Problem:**
- Visibility queries slow with 100M+ workflows
- Elasticsearch becomes bottleneck
- List operations timeout
- No query result caching
- Hot partitions in visibility store

**Proposed Solutions:**
- Materialized views for common queries
- Query result caching layer (Redis)
- Pagination improvements (keyset pagination)
- Query optimization (index hints, query rewriting)
- Sharded visibility store
- Alternative visibility backends (ClickHouse, BigQuery)

**Why Medium-High:** Impacts large deployments. Smaller deployments don't hit this.

---

### RFC-0017: Workflow Scheduling Improvements (1M+ schedules)
**Impact:** MEDIUM-HIGH | **Effort:** 6-8 weeks

**Problem:**
- Current cron scheduling doesn't scale to millions
- Schedule drift under load
- No backpressure for scheduled workflows
- No schedule dependencies
- Limited schedule management UI

**Proposed Solutions:**
- Distributed scheduler redesign
- Schedule sharding by time range
- Backpressure-aware scheduling
- Schedule priorities
- Schedule dependency graphs (DAG)
- Bulk schedule operations API

**Why Medium-High:** Many users need large-scale scheduling (ETL, batch jobs, recurring tasks).

---

### RFC-0018: API Modernization (GraphQL, Batch Operations, Streaming)
**Impact:** MEDIUM-HIGH | **Effort:** 8-10 weeks

**Problem:**
- Only gRPC/HTTP APIs (no GraphQL)
- No batch operations (must loop)
- No streaming for large result sets
- No webhook/callback support
- API versioning unclear

**Proposed Solutions:**
- GraphQL API for flexible querying
- Batch operations API (bulk start, bulk terminate)
- Streaming APIs (server-sent events, gRPC streaming)
- Webhook support for workflow events
- API versioning strategy (v2 API)

**Why Medium-High:** Developer experience. Modern APIs expected by new users.

---

### RFC-0019: Cross-Cluster Workflow Federation
**Impact:** MEDIUM | **Effort:** 10-12 weeks

**Problem:**
- Can't execute workflows across clusters
- Nexus covers some, but limited
- No hybrid cloud support
- No workflow mobility between clusters

**Proposed Solutions:**
- Federated workflow execution
- Cross-cluster task routing
- Workflow migration between clusters
- Hybrid cloud patterns (on-prem + cloud)
- Cluster discovery and routing

**Why Medium:** Needed for complex deployments, but Nexus covers basic cases.

---

## 🟢 Nice to Have

### RFC-0020: AI/ML Workflow Patterns & Integrations
**Impact:** MEDIUM | **Effort:** 4-6 weeks

**Problem:**
- LLM calls have unique retry requirements
- No semantic caching for expensive AI calls
- Token usage not tracked
- No built-in patterns for AI workflows

**Proposed Solutions:**
- AI-specific activity wrappers
- Semantic caching for LLM responses
- Token usage tracking and limits
- Circuit breaker for API rate limits
- AI workflow templates (agent loops, RAG patterns)

**Why Nice to Have:** Growing trend, but not core to Temporal's mission.

---

## Summary: Recommended RFCs to Add

**Top 3 (Critical Impact):**
1. **RFC-0011: Cost Optimization & Storage Management** - Directly impacts TCO
2. **RFC-0012: Multi-Region Active-Active** - Required for enterprise SLAs
3. **RFC-0013: Multi-Tenancy Resource Isolation** - Essential for SaaS

**Next 3 (High Impact):**
4. **RFC-0014: Workflow State Migration** - Unblocks long-running workflow improvements
5. **RFC-0015: Advanced Observability & Debugging** - Major developer productivity gain
6. **RFC-0016: Query Performance at Scale** - Critical for large deployments

**Consider if Time Permits:**
7. RFC-0017: Workflow Scheduling at Scale
8. RFC-0018: API Modernization
9. RFC-0019: Cross-Cluster Federation
10. RFC-0020: AI/ML Integration

---

## Investment Roadmap (If All Implemented)

**Phase 1 (Months 1-4): Foundation**
- Cost Optimization (RFC-0011)
- Multi-Tenancy (RFC-0013)

**Phase 2 (Months 5-8): Scale & Reliability**
- Multi-Region Active-Active (RFC-0012)
- Query Performance at Scale (RFC-0016)

**Phase 3 (Months 9-12): Developer Experience**
- Workflow State Migration (RFC-0014)
- Advanced Observability (RFC-0015)
- API Modernization (RFC-0018)

**Total Additional Investment:** ~$1.2M (80 engineering weeks across 3 phases)
**Combined with existing 10 RFCs:** ~$2M total investment roadmap
