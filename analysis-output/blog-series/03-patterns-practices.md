# Design Patterns and Practices in Temporal

**Blog Series:** Part 3 of 7 | **Commit SHA:** `0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3`

## Core Design Patterns

### 1. Event Sourcing Pattern

**Implementation:** Every workflow state change = immutable event
- **Trade-off:** Storage cost for complete audit trail and replay capability
- **Benefits:** Time-travel debugging, deterministic replay, CQRS-ready
- **Code:** [`service/history/events/`](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/service/history/events/)

### 2. CQRS (Command Query Responsibility Segregation)

**Separation:**
- **Command Side:** Execution Store (Cassandra/MySQL/PostgreSQL) - optimized for writes
- **Query Side:** Visibility Store (Elasticsearch/SQL) - optimized for searches

**Why:** Different access patterns require different optimizations

### 3. Transactional Outbox Pattern

**Problem:** How to guarantee task reaches Matching if History crashes?

**Solution:**
```sql
BEGIN TRANSACTION;
  UPDATE workflow_state
  INSERT history_event
  INSERT transfer_task  -- Outbox!
COMMIT;
```

**Guarantee:** At-least-once delivery via async queue processor

**Code:** [`service/history/queues/`](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/service/history/queues/)

### 4. Sharding Pattern

**Fixed Hash Sharding:**
- Shard count immutable (set at cluster creation)
- Workflows distributed via `hash(workflowID) % numShards`
- **Trade-off:** Simplicity vs. dynamic elasticity

**Alternative Considered:** Dynamic sharding (Kafka-style) - rejected due to complexity

### 5. Circuit Breaker Pattern

**Implementation:** `github.com/sony/gobreaker`
- **States:** Closed → Open → Half-Open
- **Purpose:** Prevent cascading failures

**Location:** [`common/rpc/interceptor/`](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/common/rpc/interceptor/)

### 6. Retry Pattern (Three Layers)

1. **gRPC Server-Side Retry:** 1 extra attempt, minimal backoff
2. **gRPC Client-Side Retry:** Exponential backoff, route to different server
3. **Application-Level Retry:** Workflow/activity retry policies

**Why Three Layers?** Defense in depth, different failure modes

### 7. Dependency Injection (Uber Fx)

**Pattern:** Constructor injection with explicit dependency graph

```go
// temporal/fx.go
fx.Provide(
    persistenceClient,
    historyEngine,
    matchingClient,
    // Fx wires dependencies automatically
)
```

**Benefits:** Testability, clear dependencies, lifecycle management

### 8. Observer Pattern (Metrics/Tracing)

**Implementation:** gRPC interceptors for cross-cutting concerns

```go
server := grpc.NewServer(
    grpc.UnaryInterceptor(
        grpc_middleware.ChainUnaryServer(
            otelgrpc.UnaryServerInterceptor(),  // Tracing
            metricsInterceptor,                 // Metrics
            rateLimitInterceptor,               // Rate limiting
        ),
    ),
)
```

---

## Error Handling Practices

### Retryable vs. Non-Retryable Errors

**Retryable:**
- `Unavailable` - Service down
- `ResourceExhausted` - Rate limited
- `Internal` - Server error (sometimes)

**Non-Retryable:**
- `InvalidArgument` - Bad request
- `NotFound` - Resource missing
- `PermissionDenied` - Auth failure

**Code:** [`common/backoff/retry.go`](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/common/backoff/retry.go)

---

## Observability Patterns

### Structured Logging

```go
logger.Info("Workflow started",
    tag.WorkflowID(workflowID),
    tag.WorkflowType(workflowType),
    tag.Attempt(attempt))
```

**Framework:** Uber Zap (zero-allocation logging)

### Metrics Collection

**Dual Framework Support:**
- Tally (legacy): StatsD/Prometheus
- OpenTelemetry (modern): OTLP, multi-backend

**Priority-Based Rate Limiting:** Different quotas for system vs. user operations

### Distributed Tracing

**Automatic:** gRPC calls instrumented via `otelgrpc`
**Manual:** Span linking for batch operations

---

## Testing Practices

**Three-Tier Strategy:**
1. **Unit Tests:** Fast, no external deps (testify + gomock)
2. **Integration Tests:** Real databases (Cassandra, MySQL, PostgreSQL, SQLite)
3. **Functional Tests:** End-to-end scenarios with TaskPoller

**Test Utilities:**
- `testvars` - Deterministic test data
- `TaskPoller` - Worker simulation
- `protorequire` - Protobuf assertions

---

## Key Takeaways

1. **Event sourcing + CQRS** - Separate write and read models
2. **Transactional outbox** - Guarantees at-least-once delivery
3. **Fixed sharding** - Simple, predictable, no rebalancing
4. **Three-layer retries** - Defense in depth for failures
5. **Observability** - Metrics, tracing, structured logging built-in

**Next:** [Blog Post 4: Extending Temporal](04-extending-integrating.md)
