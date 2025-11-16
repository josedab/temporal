# RFC-0001: Virtual Sharding for Dynamic Scalability

**Status:** Draft
**Author:** Analysis Team
**Created:** 2025-11-16
**Commit SHA:** `0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3`

---

## Summary

Introduce virtual sharding layer to enable dynamic shard count adjustment without data migration, addressing Temporal's current limitation of fixed shards at cluster creation.

---

## Motivation

### Current Problem

Shard count is **immutable** after cluster creation:
```go
// config.yaml
persistence:
  numHistoryShards: 512  # CANNOT CHANGE!
```

**Consequences:**
- Must estimate capacity upfront (often wrong)
- Under-provisioning → hot shards, performance degradation
- Over-provisioning → wasted resources, inefficient
- Scaling requires new cluster + data migration (days of downtime)

### Real-World Impact

**Scenario:** Company starts with 512 shards, expecting 50K workflows/sec
- **Year 1:** 20K workflows/sec → over-provisioned (wasted $$)
- **Year 2:** 150K workflows/sec → under-provisioned (hot shards, degraded performance)
- **Options:**
  1. Live with degraded performance
  2. Migrate to new cluster (expensive, risky)
  3. Shard data across multiple clusters (operational complexity)

**None are good!**

---

## Detailed Design

### Virtual Sharding Architecture

**Concept:** Separate **logical shards** (fixed) from **physical shards** (dynamic).

```
Current (Fixed Sharding):
WorkflowID → hash → Physical Shard (512)

Proposed (Virtual Sharding):
WorkflowID → hash → Virtual Shard (65536)
Virtual Shard → map → Physical Shard (512→1024→2048...)
```

### Implementation

#### 1. Virtual Shard Layer

```go
// New abstraction
type VirtualShardMapper interface {
    // Map virtual shard to physical shard
    GetPhysicalShard(virtualShardID int32) int32
    
    // Rebalance virtual shards across physical shards
    Rebalance(oldPhysicalCount, newPhysicalCount int32) error
}

type VirtualShardMapperImpl struct {
    virtualShardCount  int32  // Fixed (e.g., 65536)
    physicalShardCount int32  // Dynamic (e.g., 512 → 1024)
    mapping            []int32 // virtualShardID → physicalShardID
}

func (m *VirtualShardMapperImpl) GetPhysicalShard(virtualShardID int32) int32 {
    return m.mapping[virtualShardID]
}
```

#### 2. Workflow ID Hashing (No Change)

```go
// Unchanged - still hash to virtual shard
virtualShardID := hash(workflowID) % m.virtualShardCount
physicalShardID := m.GetPhysicalShard(virtualShardID)
```

#### 3. Shard Addition Process

**Step 1: Admin Command**
```bash
temporal-admin shard add --count 512
# Adds 512 new physical shards (512 → 1024)
```

**Step 2: Rebalance Virtual Shards**
```go
func (m *VirtualShardMapperImpl) Rebalance(newCount int32) error {
    // Minimal disruption rebalancing
    for virtualID := 0; virtualID < m.virtualShardCount; virtualID++ {
        oldPhysical := m.mapping[virtualID]
        
        // Rebalance: move ~50% of virtual shards to new physical shards
        if virtualID % 2 == 0 {
            // Keep on old physical shard
        } else {
            // Move to new physical shard
            newPhysical := oldPhysical + (newCount / 2)
            m.mapping[virtualID] = newPhysical
            
            // Background task: migrate data
            scheduler.MigrateVirtualShard(virtualID, oldPhysical, newPhysical)
        }
    }
    m.physicalShardCount = newCount
    return nil
}
```

**Step 3: Background Data Migration**
```go
func MigrateVirtualShard(virtualID, oldPhysical, newPhysical int32) {
    // Find all workflows in virtual shard
    workflows := db.Query("SELECT * FROM executions WHERE virtual_shard_id = ?", virtualID)
    
    for _, wf := range workflows {
        // Move to new physical shard
        db.Exec("UPDATE executions SET shard_id = ? WHERE workflow_id = ?", newPhysical, wf.ID)
        db.Exec("UPDATE history_node SET shard_id = ? WHERE workflow_id = ?", newPhysical, wf.ID)
        db.Exec("UPDATE transfer_tasks SET shard_id = ? WHERE workflow_id = ?", newPhysical, wf.ID)
        db.Exec("UPDATE timer_tasks SET shard_id = ? WHERE workflow_id = ?", newPhysical, wf.ID)
    }
}
```

**Step 4: Gradual Traffic Shift**
- New workflows → new shards immediately
- Existing workflows → migrate gradually (low-priority background task)
- No downtime!

---

## Example Usage

### Before: Fixed Sharding

```yaml
# Day 1: Cluster creation
persistence:
  numHistoryShards: 512

# Year 2: Need more capacity
# Option 1: Live with hot shards (bad performance)
# Option 2: Migrate to new cluster (downtime + risk)
```

### After: Virtual Sharding

```yaml
# Day 1: Cluster creation
persistence:
  numVirtualShards: 65536   # Fixed (never changes)
  numPhysicalShards: 512    # Can change!
```

```bash
# Year 2: Need more capacity
temporal-admin shard add --count 512
# Shards: 512 → 1024 (no downtime!)

# Year 3: Need even more
temporal-admin shard add --count 1024
# Shards: 1024 → 2048
```

---

## Implementation Plan

### Phase 1: Foundation (2 weeks)
- [ ] Implement VirtualShardMapper interface
- [ ] Add virtual_shard_id column to database schema
- [ ] Update workflow ID hashing to use virtual shards
- [ ] Write migration tool for existing clusters

### Phase 2: Rebalancing (2 weeks)
- [ ] Implement rebalancing algorithm
- [ ] Background data migration task
- [ ] Admin CLI for shard addition
- [ ] Monitoring and metrics

### Phase 3: Testing (1 week)
- [ ] Unit tests for virtual shard mapper
- [ ] Integration tests for rebalancing
- [ ] Functional tests with live migration
- [ ] Performance benchmarks

### Phase 4: Rollout (1 week)
- [ ] Documentation
- [ ] Operator runbook
- [ ] Gradual rollout (canary → production)

**Total Duration:** 6-8 weeks

---

## Backwards Compatibility

### Existing Clusters

**Option 1: Opt-in (Recommended)**
- New clusters use virtual sharding by default
- Existing clusters remain on fixed sharding
- Migration tool available for those who want to upgrade

**Option 2: Transparent Migration**
- All clusters migrated automatically
- Virtual shards = physical shards initially (1:1 mapping)
- Can add shards later

### API Compatibility

**No breaking changes:**
- Workflow ID hashing unchanged (user perspective)
- Existing APIs continue to work
- Internal changes only

---

## Alternatives Considered

### Alternative 1: Consistent Hashing (Kafka-style)

**Approach:** Use consistent hashing ring for shard assignment

**Pros:**
- Minimal data movement on rebalancing

**Cons:**
- Complex to implement correctly
- Hot spot issues (uneven distribution)
- Difficult to predict capacity

**Verdict:** Rejected - complexity outweighs benefits

### Alternative 2: Multi-Cluster Sharding

**Approach:** Run multiple Temporal clusters, shard at namespace level

**Pros:**
- No code changes needed

**Cons:**
- Operational complexity (multiple clusters)
- Cannot rebalance single namespace
- Cross-cluster workflows complex

**Verdict:** Rejected - operational burden too high

### Alternative 3: Database Partitioning

**Approach:** Use database-level partitioning

**Pros:**
- Leverage database features

**Cons:**
- Database-specific (not portable)
- Limited control over rebalancing
- Performance unpredictable

**Verdict:** Rejected - too database-specific

---

## Open Questions

1. **Virtual shard count:** 65536 too many? Too few?
   - **Answer:** 65536 allows 128x growth (512 → 65536)
   
2. **Rebalancing strategy:** All at once or gradual?
   - **Answer:** Gradual (configurable rate to avoid overwhelming DB)

3. **Migration rollback:** What if migration fails?
   - **Answer:** Keep old shard data until migration confirmed successful

4. **Cross-shard queries:** How to handle?
   - **Answer:** Visibility store handles queries (unchanged)

---

## Success Metrics

- ✅ Can add shards without downtime
- ✅ Data migration completes within SLA (e.g., 24 hours for 100K workflows)
- ✅ Performance degradation < 10% during migration
- ✅ Zero data loss during migration
- ✅ Rollback capability if issues detected

---

## Risks and Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|-----------|-----------|
| **Data loss during migration** | Critical | Low | Extensive testing, gradual rollout, rollback plan |
| **Performance regression** | High | Medium | Benchmark before/after, rebalance during low-traffic periods |
| **Database lock contention** | Medium | Medium | Batch migrations, rate limiting |
| **Operational complexity** | Medium | High | Comprehensive docs, runbooks, automation |

---

## References

- [Current sharding code](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/common/persistence/shard.go)
- [Consistent Hashing](https://www.toptal.com/big-data/consistent-hashing)
- [Kafka Partition Reassignment](https://kafka.apache.org/documentation/#basic_ops_partitionassignment)

---

**Status:** Draft - awaiting stakeholder review
**Next Steps:** Architecture review, prototype implementation
