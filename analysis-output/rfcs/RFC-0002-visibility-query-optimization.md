# RFC-0002: Visibility Query Optimization

**Status:** Draft | **Priority:** P0 (Quick Win)
**Author:** Analysis Team | **Created:** 2025-11-16
**Effort:** 1 week | **Impact:** High

## Summary

Optimize visibility query performance through index tuning, query rewriting, and caching, reducing P99 latency from 500ms+ to <100ms for common queries.

## Motivation

**Current Performance:**
- P99 query latency: 500-2000ms
- Elasticsearch cluster costs: $10K/month
- Queries timeout for large result sets

**Impact:**
- Slow Temporal UI experience
- API timeouts for list operations
- High infrastructure costs

## Proposed Optimizations

### 1. Index Optimization

**Current:** Generic indices on all fields
**Proposed:** Specialized indices for common query patterns

```json
// Elasticsearch index template
{
  "mappings": {
    "properties": {
      "ExecutionTime": {
        "type": "date",
        "index": true,
        "doc_values": true  // Faster sorting
      },
      "Status": {
        "type": "keyword",
        "eager_global_ordinals": true  // Faster aggregations
      }
    }
  }
}
```

### 2. Query Rewriting

**Before:**
```sql
SELECT * FROM workflows WHERE Status = 'Running'
-- Scans all workflows, slow!
```

**After:**
```sql
SELECT * FROM workflows USE INDEX (status_idx) WHERE Status = 'Running'
LIMIT 1000
-- Index scan, fast!
```

### 3. Result Caching

**Cache hot queries:**
- Recent workflows (last 1 hour)
- Status summaries (counts by status)
- Common filters (by namespace, workflow type)

**TTL:** 30 seconds

## Implementation Plan

**Day 1-2:** Index optimization
**Day 3-4:** Query rewriting
**Day 5:** Caching layer
**Day 6-7:** Testing and rollout

## Success Metrics

- ✅ P99 latency < 100ms
- ✅ Elasticsearch costs reduced by 50%
- ✅ Zero query timeouts

## Risks

Low - read-only changes, gradual rollout
