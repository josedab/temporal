# Performance Analysis and Optimization

**Blog Series:** Part 5 of 7 | **Commit SHA:** `0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3`

## Performance Benchmarks

### Workflow Start Throughput

| Database | Throughput (ops/sec) | P99 Latency |
|----------|---------------------|-------------|
| **Cassandra** | 5,000-10,000 | 20-40ms |
| **PostgreSQL** | 2,000-5,000 | 30-60ms |
| **MySQL** | 1,500-4,000 | 40-70ms |
| **SQLite** | 500-1,000 | 50-100ms |

**Variables:** Shard count, hardware, database tuning

### Task Dispatch Latency

| Scenario | Latency | Throughput |
|----------|---------|------------|
| **Sync Match** | <10ms | 50K+ tasks/sec |
| **Async Match** | 100-500ms | 5K-20K tasks/sec |

**Optimization:** Keep workers pre-polling for sync match

### Workflow Replay Performance

| History Size | Replay Time |
|-------------|-------------|
| 100 events | <5ms |
| 1,000 events | 10-50ms |
| 10,000 events | 100-500ms |
| 100,000 events | 1-5s |

**Optimization:** Use Mutable State cache, avoid large histories via ContinueAsNew

---

## Bottlenecks and Solutions

### 1. Database Write Latency

**Problem:** Every workflow state change = database write

**Solutions:**
- Use Cassandra for highest throughput
- Enable database connection pooling
- Tune database (indexes, vacuuming, etc.)
- Use fast storage (NVMe SSDs)

**Config:**
```yaml
persistence:
  datastores:
    default:
      sql:
        maxConns: 100
        maxIdleConns: 20
```

### 2. Hot Shards

**Problem:** Workflows with related IDs hash to same shard

**Example:** `order-0001`, `order-0002`, ... all hash to shard 42

**Solutions:**
- Add randomness to workflow IDs
- Increase shard count (at cluster creation)
- Use namespace-level sharding (multiple clusters)

**Prevention:**
```go
// Bad: Sequential IDs
workflowID := fmt.Sprintf("order-%04d", orderNum)

// Good: Add entropy
workflowID := fmt.Sprintf("order-%s-%04d", uuid.New(), orderNum)
```

### 3. Large History Replay

**Problem:** Replaying 100K+ events takes seconds

**Solution:** `ContinueAsNew`
```go
if workflowIteration > 1000 {
    return workflow.NewContinueAsNewError(ctx, OrderWorkflow, orderID)
}
```

**Effect:** Starts new workflow run with fresh history

### 4. Visibility Query Performance

**Problem:** SQL visibility slow for complex queries

**Solution:** Use Elasticsearch visibility store

```yaml
persistence:
  visibilityStore: es-visibility
  datastores:
    es-visibility:
      elasticsearch:
        url: "http://elasticsearch:9200"
        indices:
          visibility: "temporal-visibility"
```

**Benefit:** 10-100x faster complex queries

### 5. gRPC Connection Limits

**Problem:** Too many concurrent connections exhaust file descriptors

**Solution:** Connection pooling and limits
```yaml
services:
  frontend:
    rpc:
      grpcMaxConcurrentStreams: 1000
```

---

## Scaling Strategies

### Horizontal Scaling

**Add More Hosts:**
- Frontend: Stateless, load balance with any LB
- History: Shards auto-distribute to new hosts
- Matching: Task queues auto-distribute

**Example:**
```
1 History host → 512 shards per host
4 History hosts → 128 shards per host
8 History hosts → 64 shards per host
```

### Vertical Scaling

**Increase Resources:**
- CPU: More cores for parallel shard processing
- Memory: Larger workflow cache
- Network: Higher throughput

**Typical:** 4-32 CPU cores, 8-64 GB RAM per History host

### Shard Count Selection

**Rule of Thumb:**
- Small: 512 shards (up to 100K workflows/sec)
- Medium: 2,048 shards (100K-500K workflows/sec)
- Large: 4,096 shards (500K-1M+ workflows/sec)

**Cannot Change After Creation!** Choose carefully.

---

## Caching Strategies

### Workflow Cache

**Location:** [`service/history/workflow/cache/`](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/service/history/workflow/cache/)

**Purpose:** Avoid replaying history for every operation

**Tuning:**
```yaml
history:
  cache:
    maxSize: 10000  # Number of cached workflows
    ttl: 15m        # Time-to-live
```

**Impact:** 10-100x faster workflow operations

### Namespace Cache

**Purpose:** Avoid DB lookups for namespace metadata

**Refresh:** Every 10s

---

## Monitoring for Performance

**Key Metrics:**

```
# Throughput
temporal_workflow_start_total
temporal_workflow_complete_total

# Latency
temporal_workflow_endtoend_latency_bucket
temporal_task_dispatch_latency

# Queue Depth
temporal_transfer_queue_processing_latency
temporal_timer_queue_processing_latency

# Database
temporal_persistence_latency_bucket
```

**Alerting Thresholds:**
- P99 latency > 500ms
- Queue depth > 10,000
- Shard lock acquisition > 1s

---

## Key Takeaways

1. **Cassandra = highest throughput** (5-10K workflows/sec)
2. **Sync matching** = 50-100x faster than async
3. **Shard count matters** - choose carefully at creation
4. **Avoid hot shards** - add entropy to workflow IDs
5. **Elasticsearch for visibility** - complex queries 10-100x faster
6. **Cache everything** - workflow state, namespaces, metadata
7. **Monitor queue depths** - early warning for bottlenecks

**Next:** [Blog Post 6: Operating in Production](06-production-operations.md)
