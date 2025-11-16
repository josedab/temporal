# Extending and Integrating Temporal

**Blog Series:** Part 4 of 7 | **Commit SHA:** `0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3`

## Extension Points

### 1. Custom Persistence Layer

**Interface:** [`common/persistence/interfaces.go`](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/common/persistence/interfaces.go)

**Built-in Implementations:**
- Cassandra: `common/persistence/cassandra/`
- PostgreSQL: `common/persistence/sql/postgresql/`
- MySQL: `common/persistence/sql/mysql/`
- SQLite: `common/persistence/sql/sqlite/`

**How to Add Custom DB:**
1. Implement `ExecutionManager`, `ShardManager`, `TaskManager` interfaces
2. Implement schema versioning
3. Register in persistence factory

### 2. Pluggable Metrics

**Frameworks Supported:**
- Tally (Uber): StatsD, Prometheus
- OpenTelemetry: OTLP, any OTEL-compatible backend

**Configuration:**
```yaml
metrics:
  prometheus:
    framework: "opentelemetry"
    listenAddress: "0.0.0.0:8000"
```

### 3. Dynamic Configuration

**Use Cases:**
- Runtime rate limit adjustments
- Feature flags
- Timeout tuning
- Shard threshold changes

**Implementation:**
```yaml
# config/dynamicconfig/development.yaml
frontend.rps: 1000
matching.numTaskqueuePartitions: 4
```

**Polls every 10s for changes** - zero-downtime configuration updates

### 4. Authorization Hooks

**Interface:** `Authorizer` in `common/authorization/`

**Built-in:** JWT, OAuth2, mTLS

**Custom Implementation:**
```go
type Authorizer interface {
    Authorize(ctx context.Context, claims *Claims, target *CallTarget) error
}
```

### 5. Archival Providers

**Built-in:**
- AWS S3: `common/archiver/s3store/`
- Google Cloud Storage: `common/archiver/gcloud/`
- Local Filesystem: `common/archiver/filestore/`

**How to Add:** Implement `HistoryArchiver` and `VisibilityArchiver` interfaces

---

## Integration Patterns

### 1. Nexus: Cross-Namespace Orchestration

**Use Case:** Orchestrate workflows across namespaces/clusters

**Example:**
```go
// Invoke workflow in different namespace
nexus.Execute(ctx, nexus.ExecuteOptions{
    Endpoint:  "other-namespace",
    Operation: "ProcessOrder",
    Input:     orderData,
})
```

**Code:** [`components/nexusoperations/`](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/components/nexusoperations/)

### 2. Custom Search Attributes

**Define in Visibility Store:**
```go
// Add custom attributes for advanced queries
temporal operator search-attribute create \
    --name CustomStatus \
    --type Keyword
```

**Query:**
```sql
SELECT * FROM workflows WHERE CustomStatus = 'pending-approval'
```

### 3. SDK Integration

**Available SDKs:**
- Go: `go.temporal.io/sdk`
- Java: `io.temporal:temporal-sdk`
- TypeScript: `@temporalio/client`
- Python: `temporalio`
- .NET: `Temporalio`

**Custom SDK:** Implement Temporal API protocol (protobuf-based gRPC)

---

## Key Takeaways

1. **Persistence abstraction** - Swap databases without code changes
2. **Dynamic config** - Runtime tuning without restarts
3. **Pluggable observability** - Choose your metrics/tracing stack
4. **Nexus** - Federated workflow orchestration
5. **Extensible** - Clean interfaces for custom implementations

**Next:** [Blog Post 5: Performance Analysis](05-performance-analysis.md)
