# Operating Temporal in Production

**Blog Series:** Part 6 of 7 | **Commit SHA:** `0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3`

## Critical Configuration Decisions

### 1. Shard Count Selection (Immutable!)

**Choose Based on Peak Expected Load:**
- 512 shards: Up to 100K workflows/sec
- 2,048 shards: 100K-500K workflows/sec
- 4,096 shards: 500K-1M+ workflows/sec

**Cannot change later without data migration!**

### 2. Database Selection

**Production Recommendations:**
- **Best Performance:** Cassandra (multi-datacenter replication)
- **Operational Simplicity:** PostgreSQL (managed RDS/Cloud SQL)
- **Cost Optimized:** PostgreSQL (sufficient for most workloads)
- **Not Recommended:** SQLite (development only)

### 3. Visibility Store

**Small Scale (<100K workflows):** SQL visibility (same DB as execution)
**Large Scale:** Elasticsearch (better query performance)

### 4. High Availability Setup

**Minimum Production:**
- 3x Frontend (behind load balancer)
- 4x History (distribute shards)
- 2x Matching (active-active)
- 1x Worker (system workflows)
- Multi-AZ database deployment

---

## Security Hardening

### TLS Everywhere

```yaml
global:
  tls:
    frontend:
      server:
        certFile: /certs/server.crt
        keyFile: /certs/server.key
      client:
        rootCaFiles: [/certs/ca.crt]
    internode:
      server:
        certFile: /certs/internode.crt
        keyFile: /certs/internode.key
```

**Require mTLS for production!**

### Authentication & Authorization

**Options:**
- JWT tokens
- OAuth2 integration
- LDAP/Active Directory
- Custom authorizer

**Config:**
```yaml
global:
  authorization:
    jwtKeyProvider:
      keySourceURIs: ["https://auth.example.com/keys"]
```

### Secrets Management

**Do Not:**
- ❌ Hardcode credentials in config
- ❌ Store secrets in environment variables (visible in process list)

**Do:**
- ✅ Use secret management (AWS Secrets Manager, HashiCorp Vault)
- ✅ Rotate credentials regularly
- ✅ Use IAM roles (cloud environments)

---

## Monitoring & Alerting

### Critical Alerts

**Queue Processing:**
```
# Alert if queue processing falls behind
ALERT QueueProcessingLag
  IF temporal_queue_lag_seconds > 300
  FOR 5m
```

**Shard Lock Contention:**
```
ALERT ShardLockContention
  IF temporal_shard_lock_latency_seconds > 1
  FOR 2m
```

**Database Latency:**
```
ALERT DatabaseLatencyHigh
  IF temporal_persistence_latency_seconds{quantile="0.99"} > 0.5
  FOR 5m
```

### Dashboards

**Key Metrics to Track:**
- Workflow start/completion rates
- Task dispatch latency
- Queue depths (transfer, timer)
- Database connection pool usage
- gRPC connection counts
- Memory/CPU usage per service

---

## Capacity Planning

### Calculate Required Capacity

```
Shard Count = Peak Workflows/sec × 0.0001
(Assuming 10K workflows/sec per shard)

Example:
500K workflows/sec → 500K × 0.0001 = 5,000 shards
Use 4,096 or 8,192 shards
```

**History Hosts:**
```
Hosts = Shard Count / Shards per Host
(Typical: 64-128 shards per host)

Example:
4,096 shards / 64 = 64 hosts
```

**Database:**
```
IOPS = Workflows/sec × 10
(Each workflow ≈ 10 write operations)

Example:
50K workflows/sec → 500K IOPS required
```

---

## Disaster Recovery

### Backup Strategy

**What to Backup:**
1. Database (execution + visibility stores)
2. Configuration files
3. TLS certificates/keys
4. Dynamic configuration

**Frequency:**
- **Critical:** Hourly database snapshots
- **Important:** Daily full backups
- **Config:** On change

### Recovery Procedures

**Database Failure:**
1. Promote replica to primary (if multi-region)
2. Restore from latest snapshot
3. Replay WAL/oplog to latest

**Cluster Failure:**
1. Spin up new cluster
2. Restore database
3. Update DNS to new cluster
4. Resume workers

**RTO/RPO Targets:**
- RTO (Recovery Time): <15 minutes
- RPO (Recovery Point): <5 minutes

---

## Common Issues & Solutions

### Issue: Hot Shards

**Symptoms:** Single History host at 100% CPU, others idle

**Cause:** Workflow IDs with poor distribution (e.g., sequential)

**Solution:** Add entropy to workflow IDs

### Issue: Memory Leaks

**Symptoms:** Gradual memory growth, eventual OOM

**Cause:** Workflow cache not evicting

**Solution:** Tune cache TTL and size

### Issue: Database Connection Exhaustion

**Symptoms:** "Too many connections" errors

**Cause:** Connection pool misconfigured

**Solution:** Increase pool size or reduce concurrency

---

## Upgrade Strategy

**Zero-Downtime Upgrades:**
1. Deploy new version alongside old (blue-green)
2. Shift traffic gradually to new version
3. Monitor for errors
4. Decommission old version

**Database Migrations:**
```bash
# Run schema update (backward compatible)
./temporal-sql-tool --database temporal \
    update-schema \
    --schema-dir schema/postgresql/v12/temporal/versioned \
    --version 1.18
```

**Worker Versioning:** Use Temporal's built-in worker versioning (deployment feature)

---

## Key Takeaways

1. **Choose shard count carefully** - cannot change easily
2. **Enable TLS everywhere** - production requirement
3. **Monitor queue depths** - early warning system
4. **Multi-AZ deployment** - for high availability
5. **Regular backups** - disaster recovery
6. **Plan capacity** - shard count, hosts, database IOPS
7. **Test disaster recovery** - at least quarterly

**Next:** [Blog Post 7: The Future of Temporal](07-future-innovations.md)
