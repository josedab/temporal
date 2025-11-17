# Design Patterns and Practices in Temporal

**Blog Series:** Part 3 of 7
**Commit SHA:** `0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3`
**Reading Time:** ~18 minutes
**Prerequisites:** [Part 1: Architecture](01-architecture-overview.md), [Part 2: Workflow Execution](02-deep-dive-workflow-execution.md)

---

## What You'll Learn

By the end of this post, you'll understand:
- How event sourcing pattern enables Temporal's durability guarantees
- Why CQRS separates execution from visibility
- How the transactional outbox pattern ensures reliable task delivery
- Retry strategies and their implementation across three layers
- Observability patterns for production debugging
- Testing approaches and best practices

---

## Introduction: Patterns as Solutions to Recurring Problems

Design patterns aren't just academic concepts—they're battle-tested solutions to real problems. Temporal's architecture leverages several fundamental distributed systems patterns, each chosen deliberately to solve specific challenges.

Let's explore these patterns, understand the problems they solve, and examine the trade-offs made.

---

## Pattern 1: Event Sourcing

### The Problem

Traditional systems store **current state**:

```json
{
  "workflowID": "order-123",
  "status": "payment_completed",
  "paymentID": "pay_abc",
  "inventoryReserved": true,
  "lastUpdated": "2025-11-16T10:30:00Z"
}
```

**Issues:**
1. **Lost history:** Can't see how we got to current state
2. **Crash vulnerability:** If server crashes mid-update, state is inconsistent
3. **No replay:** Can't reconstruct past states
4. **No audit trail:** Compliance requirements unmet

### The Solution: Event Sourcing

Instead of storing state, store **events**:

```json
[
  {
    "eventID": 1,
    "eventType": "WorkflowExecutionStarted",
    "timestamp": "2025-11-16T10:00:00Z",
    "attributes": {"workflowType": "OrderWorkflow"}
  },
  {
    "eventID": 2,
    "eventType": "WorkflowTaskScheduled",
    "timestamp": "2025-11-16T10:00:01Z"
  },
  {
    "eventID": 3,
    "eventType": "WorkflowTaskCompleted",
    "timestamp": "2025-11-16T10:00:05Z",
    "commands": [{"type": "ScheduleActivityTask"}]
  },
  {
    "eventID": 4,
    "eventType": "ActivityTaskScheduled",
    "timestamp": "2025-11-16T10:00:05Z",
    "attributes": {"activityType": "ChargePayment"}
  },
  {
    "eventID": 5,
    "eventType": "ActivityTaskCompleted",
    "timestamp": "2025-11-16T10:00:10Z",
    "result": "pay_abc"
  }
]
```

**Current state is derived** by replaying events. See [Part 2](02-deep-dive-workflow-execution.md) for implementation details.

### Trade-offs

**Benefits:**
- ✅ **Complete audit trail** - Every state change recorded
- ✅ **Time-travel debugging** - Inspect workflow at any point
- ✅ **Deterministic replay** - Reconstruct state from events
- ✅ **CQRS-ready** - Can project events into different read models

**Costs:**
- ❌ **Storage overhead** - Events accumulate (2-3x more than mutable state)
- ❌ **Query complexity** - Can't directly query current state
- ❌ **Replay performance** - Large histories take time to replay

**Mitigations:**
- Archival to cold storage ([`common/archiver/`](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/common/archiver/))
- Mutable State cache for fast queries
- ContinueAsNew for long-running workflows
- Visibility store for queries (CQRS)

---

## Pattern 2: CQRS (Command Query Responsibility Segregation)

### The Problem

Event sourcing makes **queries expensive** - must replay all events to determine current state.

### The Solution: Separate Read and Write Models

**Command Side:** Execution Store (Cassandra/MySQL/PostgreSQL)
- Optimized for writes
- Strong consistency

**Query Side:** Visibility Store (Elasticsearch/SQL)
- Optimized for searches
- Eventually consistent

```
History Service
    ├─► Execution Store (writes)
    └─► Visibility Store (async updates)

Users query Visibility Store (fast!)
```

### Implementation

**Write path:**
```go
// Update execution store
historyService.UpdateWorkflowExecution(ctx, ...)

// Async: Update visibility store
visibilityManager.RecordWorkflowExecutionUpdated(ctx, ...)
```

**Read path:**
```go
// Complex queries hit visibility store
visibilityManager.ListWorkflowExecutions(ctx, &visibility.ListWorkflowExecutionsRequest{
    Query: "WorkflowType='OrderWorkflow' AND Status='Running'",
})
```

### Trade-offs

**Benefits:**
- ✅ **Fast queries** - Optimized indices
- ✅ **Complex filters** - Full-text search, aggregations
- ✅ **Independent scaling** - Scale reads separately from writes

**Costs:**
- ❌ **Eventually consistent** - ~100ms lag
- ❌ **Additional infrastructure** - Elasticsearch cluster
- ❌ **Sync complexity** - Keep two stores in sync

---

## Pattern 3: Transactional Outbox

### The Problem

How to reliably send a message after updating local state?

**Broken approach:**
```go
// DON'T DO THIS!
db.Update("UPDATE workflows SET status = ?", status)  // What if this succeeds...
matchingClient.AddTask(task)  // ...but this fails?
```

### The Solution

Include the message in the transaction:

```go
transaction {
    UPDATE executions SET ...
    INSERT INTO transfer_tasks VALUES (...)  // The outbox!
}

// Async processor delivers tasks reliably
```

**Guarantees:** At-least-once delivery with durability

**Location:** [`service/history/queues/`](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/service/history/queues/)

---

## Pattern 4: Three-Layer Retry Strategy

Temporal implements retries at **three distinct layers**:

### Layer 1: gRPC Server-Side Retry

**One immediate retry** for transient errors. Saves ~50ms on transient failures.

```go
// common/rpc/interceptor/retry.go
resp, err := handler(ctx, req)
if err != nil && isRetryable(err) {
    resp, err = handler(ctx, req)  // Immediate retry
}
```

### Layer 2: gRPC Client-Side Retry

**Exponential backoff** retry, can route to different server.

```go
// common/backoff/retry.go
for attempt := 1; attempt <= maxAttempts; attempt++ {
    err = operation()
    if err == nil || !isRetryable(err) {
        return err
    }
    backoff := calculateBackoff(attempt)
    time.Sleep(backoff)
}
```

**Backoff sequence:**
- Attempt 1: 100ms
- Attempt 2: 200ms
- Attempt 3: 400ms
- Attempt 4: 800ms
- Attempt 5: 1000ms (capped)

### Layer 3: Application-Level Retry

**User-configured** retry policies for workflows and activities.

```go
workflow.ExecuteActivity(ctx, workflow.ActivityOptions{
    RetryPolicy: &temporal.RetryPolicy{
        InitialInterval:    time.Second,
        BackoffCoefficient: 2.0,
        MaximumAttempts:    5,
    },
}, ChargePayment, amount)
```

### Why Three Layers?

Different failure modes require different strategies:

| Layer | Handles | Latency | When Used |
|-------|---------|---------|-----------|
| Server Retry | Transient glitches | ~0ms | Every request |
| Client Retry | Service unavailable | 100ms-10s | Outages |
| Application Retry | Business failures | Seconds-hours | Payment declined, rate limits |

---

## Pattern 5: Circuit Breaker

### The Problem

**Cascading failures:** Retrying a failing service makes it worse.

### The Solution

**Circuit breaker** stops calling failing service temporarily.

**States:**
- **Closed:** Normal operation
- **Open:** Service failing, reject immediately
- **Half-Open:** Testing recovery with limited requests

**Implementation:** [`github.com/sony/gobreaker`](https://github.com/sony/gobreaker)

```go
breaker := gobreaker.NewCircuitBreaker(gobreaker.Settings{
    MaxRequests: 3,           // Half-open: allow 3 test requests
    Timeout:     30 * time.Second,  // Open → Half-Open after 30s
    ReadyToTrip: func(counts gobreaker.Counts) bool {
        failureRatio := float64(counts.TotalFailures) / float64(counts.Requests)
        return counts.Requests >= 10 && failureRatio >= 0.5
    },
})
```

**Benefits:**
- ✅ **Fail fast** - Don't wait for timeout
- ✅ **Backpressure** - Prevent overwhelming service
- ✅ **Automatic recovery** - Test periodically

---

## Pattern 6: Dependency Injection (Uber Fx)

### The Problem

Manual dependency wiring is error-prone and hard to test.

### The Solution

**Declarative dependency graph** with Uber Fx.

**Location:** [`temporal/fx.go`](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/temporal/fx.go)

```go
var ServerFx = fx.Options(
    fx.Provide(loggerProvider),
    fx.Provide(configProvider),
    fx.Provide(metricsProvider),
    fx.Provide(persistenceClientProvider),
    fx.Provide(historyEngineProvider),
    fx.Provide(frontendHandlerProvider),
    fx.Invoke(startServices),
)
```

**Benefits:**
- ✅ **Type-safe** - Compile-time dependency checking
- ✅ **Lifecycle management** - Automatic startup/shutdown
- ✅ **Testability** - Easy to mock dependencies
- ✅ **Fail fast** - Missing dependencies detected at startup

---

## Observability Patterns

### Structured Logging

**Framework:** Uber Zap (zero-allocation logging)

```go
logger.Info("Workflow started",
    tag.WorkflowID(workflowID),
    tag.WorkflowType(workflowType),
    tag.Namespace(namespace))

// Output (JSON):
{
  "level": "info",
  "ts": "2025-11-16T10:00:00.000Z",
  "msg": "Workflow started",
  "wf-id": "order-123",
  "wf-type": "OrderWorkflow",
  "namespace": "production"
}
```

### Distributed Tracing

**Automatic** via gRPC interceptors:

```go
import "go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc"

server := grpc.NewServer(
    grpc.StatsHandler(otelgrpc.NewServerHandler()),
)
```

Every gRPC call automatically traced!

### Metrics Collection

**Dual framework support:**
- **Tally** (legacy): StatsD, Prometheus
- **OpenTelemetry** (modern): OTLP, multi-backend

```go
// common/metrics/otel_metrics_handler.go
counter := metricsHandler.Counter("temporal_workflow_start")
counter.Record(1, metrics.NamespaceTag(namespace))
```

---

## Testing Patterns

### Three-Tier Testing Strategy

**Tier 1: Unit Tests** - Fast, no external dependencies
```go
func TestMutableState_AddActivityTaskScheduledEvent(t *testing.T) {
    ms := NewMutableState(...)
    event, err := ms.AddActivityTaskScheduledEvent(...)
    require.NoError(t, err)
}
```

**Tier 2: Integration Tests** - Real databases
```go
func (s *ExecutionManagerSuite) TestUpdateWorkflowExecution() {
    // Tests against Cassandra/MySQL/PostgreSQL/SQLite
    _, err := s.ExecutionManager.UpdateWorkflowExecution(ctx, request)
    s.NoError(err)
}
```

**Tier 3: Functional Tests** - End-to-end
```go
func (s *FunctionalSuite) TestActivityRetry() {
    we, err := s.client.ExecuteWorkflow(ctx, ...)
    poller := taskpoller.New(s.client, "task-queue")
    task := poller.PollWorkflowTask()
    // ... verify behavior
}
```

### Test Utilities

**Location:** [`common/testing/`](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/common/testing/)

- **testvars** - Deterministic test data
- **TaskPoller** - Worker simulation
- **protorequire** - Protobuf assertions
- **historyrequire** - History event assertions

---

## Key Takeaways

1. **Event Sourcing** - Complete audit trail at cost of storage
2. **CQRS** - Separate write-optimized from query-optimized stores
3. **Transactional Outbox** - Reliable message delivery
4. **Three-layer retries** - Defense in depth
5. **Circuit Breaker** - Prevent cascading failures
6. **Dependency Injection** - Type-safe wiring with Fx
7. **Observability** - Structured logging, tracing, metrics
8. **Testing** - Three tiers ensure quality

**Next:** [Blog Post 4: Extending Temporal](04-extending-integrating.md)

---

## Further Reading

- [Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html) - Martin Fowler (2005)
- [CQRS](https://martinfowler.com/bliki/CQRS.html) - Martin Fowler (2011)
- [Transactional Outbox](https://microservices.io/patterns/data/transactional-outbox.html) - Chris Richardson
- [Circuit Breaker](https://martinfowler.com/bliki/CircuitBreaker.html) - Martin Fowler (2014)
- [Temporal Architecture Docs](https://github.com/temporalio/temporal/tree/main/docs/architecture)

---

**Questions?** [Join Temporal Slack](https://t.mp/slack) or [GitHub Discussions](https://github.com/temporalio/temporal/discussions)
