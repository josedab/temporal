# Temporal Server Terminology Glossary

**Analysis Date:** 2025-11-16
**Commit SHA:** `0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3`

---

## Overview

This glossary defines Temporal-specific terminology used throughout the codebase, documentation, and architecture discussions.

---

## Core Concepts

### Activity
**Definition:** A single, well-defined action performed by a worker (typically I/O operations like API calls, database queries, file operations).

**Characteristics:**
- Non-deterministic (can have side effects)
- Executed in worker processes
- Can be retried automatically
- Has timeout settings

**Example:**
```go
func SendEmail(ctx context.Context, to string, body string) error {
    // Activity implementation
}
```

**Related Terms:** Activity Task, Activity Heartbeat, Activity Timeout

---

### Workflow
**Definition:** A fault-tolerant, durable function that orchestrates activities and other workflows.

**Characteristics:**
- Deterministic (must produce same result on replay)
- Cannot make direct I/O calls
- Survives process failures
- Can run for months/years

**Example:**
```go
func OrderProcessingWorkflow(ctx workflow.Context, orderID string) error {
    // Workflow implementation
}
```

**Related Terms:** Workflow Execution, Workflow Task, Workflow History

---

### Workflow Execution
**Definition:** A single run of a workflow, identified by a unique WorkflowID and RunID.

**Components:**
- **WorkflowID:** User-defined identifier (unique within namespace)
- **RunID:** System-generated UUID (unique globally)
- **State:** Current state (running, completed, failed, etc.)
- **History:** Complete event log

**Lifecycle:** Started → Running → Completed/Failed/Terminated/Cancelled/Timed Out

---

### Namespace
**Definition:** A logical multi-tenancy unit that isolates workflows and activities.

**Purpose:**
- Resource isolation
- Configuration scoping
- Access control boundary
- Metrics/logging separation

**System Namespaces:**
- `temporal-system` - System workflows (archival, cleanup)

**Example:**
```bash
temporal operator namespace create my-app-production
```

---

### Task Queue
**Definition:** A named queue that routes workflow tasks and activity tasks to workers.

**Types:**
1. **Workflow Task Queue** - Routes workflow tasks
2. **Activity Task Queue** - Routes activity tasks

**Characteristics:**
- Named (e.g., "order-processing-queue")
- Can be partitioned for load balancing
- Supports worker versioning

**Example:**
```go
worker.Start(taskQueue: "order-processing")
```

**Related Terms:** Sticky Task Queue, Task Queue Partition

---

## Event Sourcing Terms

### History Event
**Definition:** An immutable record of a state change in a workflow execution.

**Examples:**
- `WorkflowExecutionStarted`
- `ActivityTaskScheduled`
- `ActivityTaskCompleted`
- `WorkflowExecutionCompleted`

**Properties:**
- Sequential (EventID: 1, 2, 3, ...)
- Immutable (append-only)
- Persisted to database

---

### Mutable State
**Definition:** Denormalized, in-memory representation of a workflow's current state.

**Purpose:**
- Fast access without replaying history
- Contains: pending activities, timers, child workflows
- Rebuilt from history events if evicted from cache

**Location:** `service/history/workflow/mutable_state_impl.go`

---

### Deterministic Replay
**Definition:** Process of reconstructing workflow state by re-executing workflow code against history events.

**Why Important:**
- Enables workflow code updates (versioning)
- Allows recovery after server restarts
- Powers workflow debugging

**Non-Determinism Errors:**
- Random number generation
- Current time access (without workflow.Now())
- Direct I/O operations

---

## Service Architecture Terms

### Frontend Service
**Definition:** User-facing service handling all external API requests.

**Responsibilities:**
- StartWorkflowExecution, SignalWorkflow, QueryWorkflow
- Namespace management
- Rate limiting
- Request routing to History service

**Port:** 7233 (gRPC), 7243 (HTTP)

---

### History Service
**Definition:** Core service managing workflow execution state and event history.

**Responsibilities:**
- Event sourcing and storage
- Workflow state machine
- Shard ownership
- Task generation (transfer, timer)
- Replication

**Sharding:** Fixed shard count, workflows hashed by WorkflowID

---

### Matching Service
**Definition:** Service managing task queues and dispatching tasks to workers.

**Responsibilities:**
- Task queue management
- Worker polling (long-poll)
- Sync matching (immediate dispatch)
- Backlog tracking

**Sync Matching:** If worker is waiting when task arrives, dispatch immediately (<10ms latency)

---

### Worker Service
**Definition:** Internal service running system workflows and background processing.

**Responsibilities:**
- Cross-datacenter replication
- Scheduled workflows
- System maintenance workflows
- Archival

**Note:** Not to be confused with user workers

---

## Sharding & Partitioning

### Shard
**Definition:** A partition of workflow executions managed by a single History service instance.

**Characteristics:**
- Fixed count (set at cluster creation, e.g., 512 or 4096)
- Workflows hashed to shard by WorkflowID
- Owned by one History host at a time

**RangeID:** Monotonic counter for shard ownership (prevents split-brain)

**Example:**
```
WorkflowID "order-12345" → Hash → Shard 127
```

---

### History Shard
**Definition:** Logical partition containing subset of workflow executions.

**Components:**
- Execution data (mutable state, history events)
- Transfer tasks (schedule activity, start child workflow)
- Timer tasks (workflow timeout, activity timeout)
- Queue states (ack levels, processing state)

**Ownership:** Single History service instance owns each shard (via membership protocol)

---

## Task Types

### Workflow Task
**Definition:** A task for a worker to advance the workflow state machine.

**Triggers:**
- Workflow start
- Activity completion
- Signal received
- Timer fired

**Worker Response:** List of commands (ScheduleActivityTask, StartChildWorkflow, CompleteWorkflow)

**Location:** Dispatched via Matching service

---

### Activity Task
**Definition:** A task for a worker to execute an activity.

**Properties:**
- Contains activity input
- Has timeout settings
- Can heartbeat for long-running operations

**Worker Response:** Activity result or error

---

### Transfer Task
**Definition:** Internal task for History service to perform asynchronous operations.

**Examples:**
- Enqueue workflow task to Matching
- Enqueue activity task to Matching
- Persist visibility record

**Processing:** Asynchronous via queue processor

---

### Timer Task
**Definition:** Internal task for History service to handle time-based events.

**Examples:**
- Workflow execution timeout
- Activity task timeout
- Workflow sleep (timer command)
- Cron schedule trigger

**Processing:** Time-based queue processor

---

## Persistence Terms

### Execution Store
**Definition:** Database storing workflow execution data.

**Tables/Collections:**
- `executions` - Workflow execution metadata and mutable state
- `history_node` - History event batches
- `history_tree` - History branching (for reset)
- `transfer_tasks` - Transfer task queue
- `timer_tasks` - Timer task queue

---

### Visibility Store
**Definition:** Database optimized for searching and listing workflows.

**Implementations:**
- **Elasticsearch** - Advanced queries, custom search attributes
- **SQL** - Basic queries (PostgreSQL, MySQL, SQLite)
- **Cassandra** - Basic queries (deprecated for visibility)

**Use Cases:**
- `temporal workflow list`
- Advanced visibility queries
- Monitoring dashboards

---

## Advanced Concepts

### CHASM
**Definition:** Coordinated Heterogeneous Asynchronous State Machines

**Purpose:** Framework for building complex, coordinated state machines.

**Usage:**
- Scheduler workflows
- Advanced workflow patterns
- Multi-state coordination

**Location:** `/chasm` directory

---

### Nexus
**Definition:** Cross-namespace and cross-cluster orchestration framework.

**Features:**
- Invoke workflows in different namespaces
- Invoke workflows in different clusters
- Callback-based execution
- HTTP endpoints

**Status:** Evolving feature

---

### RangeID
**Definition:** Monotonic counter for shard ownership, used for fencing.

**Purpose:** Prevent split-brain scenarios where two History instances think they own the same shard.

**Mechanism:**
1. History instance acquires shard with RangeID=N
2. All operations require RangeID=N
3. If another instance acquires shard, RangeID increments to N+1
4. Operations with RangeID=N fail (old owner fenced out)

---

### Sticky Execution
**Definition:** Workflow tasks routed to the same worker that last processed them.

**Benefits:**
- Workflow state cached in worker
- No replay needed
- Lower latency

**Implementation:** Separate sticky task queue with short timeout

---

### Deployment
**Definition:** Worker versioning feature for rolling deployments.

**Purpose:**
- Route tasks to compatible workers
- Support gradual rollouts
- Prevent incompatible code versions

**Components:**
- Build ID
- Deployment rules
- Version sets

---

## Operational Terms

### Archival
**Definition:** Moving old workflow history to long-term storage (S3, GCS).

**Benefits:**
- Reduce primary database size
- Lower storage costs
- Preserve history for compliance

**Configuration:**
```yaml
archival:
  history:
    state: "enabled"
    provider:
      filestore:
        fileMode: "0666"
```

---

### Dynamic Config
**Definition:** Configuration that can be changed at runtime without server restart.

**Examples:**
- Rate limits
- Timeouts
- Feature flags
- Shard count thresholds

**Implementation:** File-based with polling, or database-backed

---

### Ringpop
**Definition:** Gossip-based membership protocol (based on SWIM).

**Purpose:**
- Service discovery
- Shard ownership determination
- Health monitoring

**Origin:** Uber OSS, forked by Temporal

---

### Membership
**Definition:** Cluster membership information (which hosts are running which services).

**Components:**
- Host list
- Service roles (frontend, history, matching, worker)
- Shard ownership (for History service)

**Protocol:** Ringpop-based gossip

---

## Metrics & Observability Terms

### OTEL
**Definition:** OpenTelemetry - unified observability framework.

**Signals:**
- Traces (distributed tracing)
- Metrics (counters, gauges, histograms)
- Logs (structured logging)

**Exporters:** OTLP, Prometheus, Jaeger, Zipkin

---

### Tally
**Definition:** Uber's metrics library (legacy, being replaced by OpenTelemetry).

**Reporters:**
- StatsD
- Prometheus
- M3

**Status:** Still supported but OpenTelemetry preferred

---

## Testing Terms

### TestVars
**Definition:** Test utility for generating consistent test variables.

**Purpose:**
- Deterministic test data
- Avoid test flakiness from random values
- Easy variant generation

**Location:** `common/testing/testvars/`

---

### TaskPoller
**Definition:** Test utility simulating worker polling for tasks.

**Purpose:**
- Functional testing without real workers
- Control task execution timing
- Verify task content

**Location:** `common/testing/taskpoller/`

---

## Protocol & API Terms

### Message Protocol
**Definition:** Transient messaging system for bidirectional workflow communication.

**Use Cases:**
- Workflow Update feature
- Request/response patterns
- Temporary messages (not in history if rejected)

**Status:** Recent addition (2024)

---

### Command
**Definition:** Decision/action returned by workflow task handler.

**Examples:**
- `ScheduleActivityTask`
- `StartTimer`
- `StartChildWorkflowExecution`
- `CompleteWorkflowExecution`

**Previously:** Called "Decisions" in older versions

---

### Signal
**Definition:** Asynchronous message sent to a running workflow.

**Characteristics:**
- Non-blocking for sender
- Workflow must handle signal
- Can carry payload
- Stored in history

**Example:**
```go
temporal workflow signal --workflow-id order-123 --name "approve"
```

---

### Query
**Definition:** Synchronous read of workflow state.

**Characteristics:**
- Read-only (cannot modify state)
- Immediate response
- Not recorded in history
- Requires sticky execution for performance

**Example:**
```go
temporal workflow query --workflow-id order-123 --name "getStatus"
```

---

### Update
**Definition:** Synchronous write to a running workflow.

**Characteristics:**
- Request/response pattern
- Can modify state
- Validation phase (can reject)
- Uses message protocol

**Status:** Recent feature (2024)

---

## Error & Retry Terms

### Retryable Error
**Definition:** Error type that triggers automatic retry.

**gRPC Codes:**
- `Unavailable` - Service temporarily down
- `ResourceExhausted` - Rate limited
- `Internal` - Server error (sometimes)

**Non-Retryable:**
- `InvalidArgument`
- `NotFound`
- `PermissionDenied`

---

### Backoff
**Definition:** Delay between retry attempts (usually exponential).

**Configuration:**
```go
RetryPolicy{
    InitialInterval:    1 * time.Second,
    BackoffCoefficient: 2.0,
    MaximumInterval:    100 * time.Second,
    MaximumAttempts:    10,
}
```

---

### Circuit Breaker
**Definition:** Pattern to prevent cascading failures by stopping requests to failing service.

**States:**
- Closed (normal operation)
- Open (blocking requests)
- Half-Open (testing recovery)

**Implementation:** `github.com/sony/gobreaker`

---

## Abbreviations & Acronyms

| Term | Full Name | Meaning |
|------|-----------|---------|
| **CDC** | Change Data Capture | Database replication technique |
| **CQRS** | Command Query Responsibility Segregation | Separate read/write models |
| **DI** | Dependency Injection | Design pattern (Uber Fx) |
| **E2E** | End-to-End | Full system testing |
| **gRPC** | gRPC Remote Procedure Call | RPC framework |
| **LRU** | Least Recently Used | Cache eviction policy |
| **NDC** | National Data Center | Multi-datacenter within region |
| **OTEL** | OpenTelemetry | Observability standard |
| **RBAC** | Role-Based Access Control | Authorization model |
| **RPC** | Remote Procedure Call | Inter-service communication |
| **SDK** | Software Development Kit | Client libraries |
| **SWIM** | Scalable Weakly-consistent Infection-style process group Membership | Gossip protocol |
| **TLS** | Transport Layer Security | Encryption protocol |
| **XDC** | Cross Data Center | Multi-cluster replication |

---

## Related Resources

### Official Documentation
- [Temporal Glossary](https://docs.temporal.io/glossary)
- [Temporal Concepts](https://docs.temporal.io/concepts)

### Architecture Docs
- [Architecture Overview](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/docs/architecture/README.md)
- [History Service](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/docs/architecture/history-service.md)

---

**Next:** [Back to Quick Start](00-quick-start.md) | [Blog Series](../blog-series/00-series-outline.md)
