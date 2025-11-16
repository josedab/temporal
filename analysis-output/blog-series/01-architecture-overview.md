# Understanding Temporal: Architecture and Core Concepts

**Blog Series:** Part 1 of 7
**Analysis Date:** 2025-11-16
**Commit SHA:** `0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3`
**Reading Time:** ~12 minutes

---

## What You'll Learn

By the end of this post, you'll understand:
- What Temporal is and the problem it solves
- The core architecture and how services work together
- Why event sourcing is fundamental to Temporal
- The key abstractions: Workflows, Activities, and Task Queues
- Trade-offs in Temporal's architectural decisions

---

## The Problem: Distributed Application Complexity

Imagine you're building an e-commerce system. A simple order flow involves:

1. Charge the customer's credit card
2. Reserve inventory
3. Send order to warehouse
4. Send confirmation email
5. Update analytics

Easy enough, right? But what happens when:
- The payment processor is temporarily down (retry?)
- The inventory service fails after charging the card (compensate?)
- The warehouse system takes 3 days to respond (wait? timeout?)
- Your service restarts mid-flow (where were we?)

You could handle each scenario manually with retries, timeouts, saga patterns, and state machines. Or you could use Temporal.

---

## What is Temporal?

**Temporal is a durable execution platform.** It runs your application logic in a fault-tolerant, resumable way that automatically handles:

- **Failures and retries** - Intermittent errors are retried automatically
- **Long-running processes** - Workflows can run for days, weeks, or months
- **State persistence** - Your progress is never lost, even on server failures
- **Distributed transactions** - Implements saga patterns and compensations
- **Visibility** - See all running workflows, their state, and history

### Real-World Use Cases

- **E-commerce** - Order processing, payment reconciliation
- **DevOps** - CI/CD pipelines, infrastructure provisioning
- **Finance** - Transaction processing, compliance workflows
- **ML/AI** - Training pipelines, model deployment
- **IoT** - Device onboarding, firmware updates

---

## Core Concepts: The Building Blocks

Before we dive into architecture, let's understand the key abstractions.

### Workflows: Orchestration Logic

A **Workflow** is a fault-tolerant function that orchestrates your business logic.

```go
// service/worker/scanner/workflow.go (simplified example)
func OrderProcessingWorkflow(ctx workflow.Context, orderID string) error {
    // This workflow is durable - it survives failures
    
    // Step 1: Charge payment (activity = external I/O)
    var paymentID string
    err := workflow.ExecuteActivity(ctx, ChargePayment, orderID).Get(ctx, &paymentID)
    if err != nil {
        return err  // Will retry based on retry policy
    }
    
    // Step 2: Reserve inventory
    err = workflow.ExecuteActivity(ctx, ReserveInventory, orderID).Get(ctx, nil)
    if err != nil {
        // Compensate: refund payment
        _ = workflow.ExecuteActivity(ctx, RefundPayment, paymentID).Get(ctx, nil)
        return err
    }
    
    // Step 3: Wait for warehouse (could take days!)
    var shipmentID string
    err = workflow.ExecuteActivity(ctx, ShipOrder, orderID).Get(ctx, &shipmentID)
    if err != nil {
        // Compensate: release inventory and refund
        _ = workflow.ExecuteActivity(ctx, ReleaseInventory, orderID).Get(ctx, nil)
        _ = workflow.ExecuteActivity(ctx, RefundPayment, paymentID).Get(ctx, nil)
        return err
    }
    
    // Step 4: Send confirmation
    _ = workflow.ExecuteActivity(ctx, SendEmail, orderID).Get(ctx, nil)
    
    return nil
}
```

**Key Properties:**
- **Deterministic** - Same inputs = same outputs (enables replay)
- **Durable** - Survives process crashes, network failures
- **Long-running** - Can sleep for months using timers
- **Versionable** - Can update code while workflows are running

### Activities: Side Effects

An **Activity** is a single, well-defined action (usually involving I/O).

```go
func ChargePayment(ctx context.Context, orderID string) (string, error) {
    // Activities can make external calls
    paymentID, err := stripeClient.Charge(orderID)
    if err != nil {
        return "", err  // Temporal will retry this
    }
    return paymentID, nil
}
```

**Key Properties:**
- **Non-deterministic** - Can have side effects (API calls, DB writes)
- **Retryable** - Automatic retries on failure
- **Timeouts** - Configurable timeouts prevent hanging forever
- **Heartbeats** - Long-running activities can report progress

### Task Queues: Work Distribution

A **Task Queue** routes work to workers (processes that execute your code).

```go
// Worker subscribes to task queue
worker := worker.New(client, "order-processing", worker.Options{})
worker.RegisterWorkflow(OrderProcessingWorkflow)
worker.RegisterActivity(ChargePayment)
worker.Start()
```

When you start a workflow:
```go
client.ExecuteWorkflow(ctx, client.StartWorkflowOptions{
    TaskQueue: "order-processing",  // Which worker pool should execute this?
}, OrderProcessingWorkflow, "order-123")
```

Temporal routes the work to any worker listening on that queue.

---

## The Secret Sauce: Event Sourcing

Here's where Temporal gets interesting. Instead of storing the **current state** of a workflow, Temporal stores the **complete history of events**.

### Example: Tracking Workflow State

**Traditional Approach** (mutable state):
```json
{
  "orderID": "order-123",
  "status": "payment_charged",
  "paymentID": "pay_abc",
  "updatedAt": "2025-11-16T10:30:00Z"
}
```

**Problem:** If the server crashes after charging but before updating status, you've lost the state.

**Temporal's Approach** (event sourcing):
```json
[
  {"eventID": 1, "type": "WorkflowExecutionStarted", "timestamp": "..."},
  {"eventID": 2, "type": "ActivityTaskScheduled", "activityType": "ChargePayment"},
  {"eventID": 3, "type": "ActivityTaskCompleted", "result": "pay_abc"},
  {"eventID": 4, "type": "ActivityTaskScheduled", "activityType": "ReserveInventory"},
  ...
]
```

**Benefit:** The complete history is **immutable** and **append-only**. You can always reconstruct the current state by replaying events.

### Why This Matters

1. **Crash Recovery** - Server crashes? Replay events to rebuild state.
2. **Time Travel Debugging** - Inspect workflow at any point in history.
3. **Deterministic Replay** - Re-run workflow code against history to verify correctness.
4. **Audit Trail** - Complete record of what happened and when.

**Trade-off:** More storage required (every state change is persisted).

**Mitigation:** Archival system moves old history to cold storage (S3, GCS).

See [service/history/events/](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/service/history/events/) for event generation code.

---

## Architecture Overview: The Four Services

Temporal server consists of four internal services that work together:

```
┌─────────────────────────────────────────────────┐
│                  Your Workers                   │
│  (Run your workflow and activity code)          │
└──────────────┬──────────────────────────────────┘
               │ gRPC
               ▼
┌─────────────────────────────────────────────────┐
│            Frontend Service                     │
│  • User-facing API                              │
│  • Authentication & authorization               │
│  • Rate limiting                                │
│  • Request routing                              │
└──────────────┬──────────────────────────────────┘
               │
         ┌─────┴──────────────────┐
         │                        │
    ┌────▼────────┐     ┌────────▼─────────┐
    │History      │◄────┤Matching Service  │
    │Service      │     │                  │
    │             │     │• Task queues     │
    │• Workflow   │     │• Worker polling  │
    │  execution  │     │• Task dispatch   │
    │• Events     │     │                  │
    │• State      │     └──────────────────┘
    │• Timers     │
    └──────┬──────┘
           │
    ┌──────▼──────────────────┐
    │  Persistence Layer      │
    │  (Cassandra/MySQL/      │
    │   PostgreSQL/SQLite)    │
    └─────────────────────────┘
```

### Frontend Service

**Location:** [`service/frontend/`](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/service/frontend/)

**Responsibilities:**
- Handle all user API calls (StartWorkflowExecution, SignalWorkflow, QueryWorkflow, etc.)
- Route requests to appropriate History shards
- Rate limiting and quota enforcement
- Authentication and authorization
- Namespace management

**Entry Point:** [`handler.go:88`](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/service/frontend/handler.go#L88)

**API Surface:**
```go
type Handler interface {
    StartWorkflowExecution(ctx context.Context, request *workflowservice.StartWorkflowExecutionRequest) (*workflowservice.StartWorkflowExecutionResponse, error)
    SignalWorkflowExecution(ctx context.Context, request *workflowservice.SignalWorkflowExecutionRequest) (*workflowservice.SignalWorkflowExecutionResponse, error)
    // ... 50+ more methods
}
```

**Why Separate?**
- Scales independently (stateless, easy to load balance)
- Single entry point for security policies
- Shields internal services from untrusted traffic

### History Service

**Location:** [`service/history/`](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/service/history/)

**Responsibilities:**
- Maintain workflow execution state
- Append history events to persistence
- Process workflow logic (state transitions)
- Generate transfer tasks (schedule activities, child workflows)
- Generate timer tasks (timeouts, sleep timers)
- Cross-datacenter replication

**Core Component:** [`historyEngine.go`](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/service/history/historyEngine.go)

**Key Concept: Sharding**

Workflows are distributed across **fixed number of shards** (e.g., 512 or 4096).

```go
// Shard calculation (simplified)
shardID := hash(workflowID) % numShards
```

Each shard is owned by one History service instance. Shards enable:
- **Horizontal scaling** - Add more History service instances
- **Parallel processing** - Each shard processes independently
- **Fault isolation** - Failure in one shard doesn't affect others

**Trade-off:** Shard count is **fixed at cluster creation** and can't easily be changed.

**Why Fixed Shards?**

Benefits:
- ✅ Simple ownership model (no rebalancing)
- ✅ Predictable performance
- ✅ Easy capacity planning

Costs:
- ❌ Must estimate capacity upfront
- ❌ Can't dynamically add shards

**Alternative Considered:** Dynamic sharding (like Kafka) was deemed too complex for the benefits.

### Matching Service

**Location:** [`service/matching/`](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/service/matching/)

**Responsibilities:**
- Manage task queues (in-memory)
- Handle worker polling (long-poll)
- Dispatch tasks to workers
- Implement "sync matching" (immediate dispatch)
- Track task queue backlog

**Core Component:** [`matcher.go`](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/service/matching/matcher.go)

**Sync Matching Magic:**

```
Traditional Queue (100-500ms latency):
1. History adds task to queue
2. Task persisted to database
3. Worker polls queue
4. Worker receives task
5. Worker executes task

Temporal Sync Match (<10ms latency):
1. History adds task to Matching
2. Worker is already waiting (long-poll)
3. Matching immediately dispatches task
4. Task persisted asynchronously

Result: 50-100x faster task dispatch!
```

See [`fairness.md`](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/service/matching/fairness.md) for multi-partition fairness algorithm.

### Worker Service

**Location:** [`service/worker/`](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/service/worker/)

**Responsibilities:**
- Cross-datacenter replication
- System workflows (archival, cleanup)
- Scheduled workflows
- Callback processing (Nexus)

**Key Insight:** The Worker Service is itself a Temporal worker running system workflows. Meta!

---

## How It All Works Together

Let's trace a workflow execution through the system:

### 1. Start Workflow

```
Client → Frontend: StartWorkflowExecution("order-123")
Frontend → History(shard_42): StartWorkflowExecution
History:
  - Append event: WorkflowExecutionStarted
  - Create transfer task: Schedule workflow task
  - Persist to database
History → Matching: Add workflow task to queue
Matching: Task added to "order-processing" queue
Frontend → Client: WorkflowID + RunID
```

### 2. Worker Polls for Task

```
Worker → Matching: PollWorkflowTaskQueue("order-processing")
Matching: Worker is now waiting (long-poll, 60s timeout)

(Meanwhile, History added task - sync match!)

Matching → Worker: Here's your workflow task!
Worker: Receives task (<10ms latency!)
```

### 3. Worker Executes Workflow Code

```go
Worker:
  - Loads workflow code
  - Executes OrderProcessingWorkflow
  - Workflow schedules activity: ChargePayment
  - Returns commands: [ScheduleActivityTask]
```

### 4. Complete Workflow Task

```
Worker → Frontend: RespondWorkflowTaskCompleted(commands)
Frontend → History: Process commands
History:
  - Append event: ActivityTaskScheduled
  - Create transfer task: Add activity task to Matching
  - Persist to database
History → Matching: Add activity task
Matching: Task available on "order-processing" queue
```

### 5. Worker Polls and Executes Activity

```
Worker → Matching: PollActivityTaskQueue
Matching → Worker: Activity task
Worker:
  - Executes ChargePayment activity
  - Calls Stripe API (side effect!)
  - Returns result: "pay_abc"
Worker → Frontend: RespondActivityTaskCompleted(result)
Frontend → History: Activity completed
History:
  - Append event: ActivityTaskCompleted
  - Create workflow task (workflow needs to react)
  - Persist to database
```

### 6. Repeat Until Workflow Completes

The cycle continues: workflow task → make decisions → schedule activities → complete activities → workflow task...

Finally:
```
Workflow code: return nil  // Workflow complete!
Worker → History: CompleteWorkflowExecution
History:
  - Append event: WorkflowExecutionCompleted
  - Clean up resources
  - Archive history (if configured)
```

Full flow visualization in [Blog Post 2](02-deep-dive-workflow-execution.md).

---

## Key Architectural Decisions and Trade-offs

### Decision 1: Event Sourcing

**What:** Store complete event history instead of current state

**Trade-offs:**
- ✅ Complete audit trail
- ✅ Time-travel debugging
- ✅ Deterministic replay
- ❌ Higher storage costs
- ❌ Cannot directly query state

**Verdict:** Worth it for durability guarantees

### Decision 2: gRPC over REST

**What:** Use gRPC for all communication (internal and SDK-to-server)

**Trade-offs:**
- ✅ 10-100x better throughput
- ✅ Bi-directional streaming
- ✅ Strong typing
- ❌ Steeper learning curve
- ❌ Less tooling than REST

**Verdict:** Performance wins for high-throughput system

See [`common/rpc/`](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/common/rpc/) for gRPC interceptors.

### Decision 3: Fixed Shards

**What:** Shard count fixed at cluster creation

**Trade-offs:**
- ✅ Simple, predictable
- ✅ No rebalancing overhead
- ❌ Must estimate capacity upfront
- ❌ Can't dynamically scale shards

**Verdict:** Simplicity over elasticity (horizontal scaling via more hosts)

### Decision 4: Service-Oriented Architecture

**What:** Separate services (Frontend, History, Matching, Worker)

**Trade-offs:**
- ✅ Independent scaling
- ✅ Fault isolation
- ✅ Clear boundaries
- ❌ More RPC overhead
- ❌ Operational complexity

**Verdict:** Worth it for large-scale deployments

See [Blog Post 3](03-patterns-practices.md) for deeper analysis.

---

## Observability: How to See What's Happening

Temporal provides excellent observability:

### Metrics

**Frameworks:** Tally (legacy) and OpenTelemetry (modern)

```yaml
# config/development.yaml
global:
  metrics:
    prometheus:
      framework: "opentelemetry"  # or "tally"
      listenAddress: "0.0.0.0:8000"
```

**Key Metrics:**
- `temporal_workflow_start_total` - Workflows started
- `temporal_workflow_endtoend_latency` - End-to-end latency
- `temporal_task_dispatch_latency` - Task dispatch time
- `temporal_persistence_latency` - Database latency

See [`common/metrics/defs.go`](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/common/metrics/defs.go) for all metric definitions.

### Tracing

**Framework:** OpenTelemetry with automatic gRPC instrumentation

```go
// Tracing automatically propagates through gRPC calls
import "go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc"

server := grpc.NewServer(
    grpc.StatsHandler(otelgrpc.NewServerHandler()),
)
```

**View Traces:** Grafana Tempo, Jaeger, Honeycomb

See [docs/development/tracing.md](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/docs/development/tracing.md)

### Logging

**Framework:** Uber Zap (high-performance structured logging)

```go
// common/log/
logger.Info("Workflow started",
    tag.WorkflowID(workflowID),
    tag.WorkflowType(workflowType),
    tag.Namespace(namespace))
```

**Output:** JSON or console format

```json
{
  "level": "info",
  "ts": "2025-11-16T10:30:00.000Z",
  "msg": "Workflow started",
  "workflow-id": "order-123",
  "workflow-type": "OrderProcessingWorkflow",
  "namespace": "production"
}
```

---

## Getting Started: Run Temporal Locally

```bash
# Prerequisites: Go 1.25+, Docker (for dependencies)

# Clone the repo
git clone https://github.com/temporalio/temporal.git
cd temporal

# Build
make

# Start Temporal (SQLite in-memory, no Docker needed)
make start

# In another terminal, create namespace
temporal operator namespace create default

# Access Web UI
open http://localhost:8080
```

Run your first workflow:
```bash
cd samples/hello
go run hello.go
```

See [CONTRIBUTING.md](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/CONTRIBUTING.md) for full setup.

---

## Key Takeaways

1. **Event sourcing is fundamental** - Every state change is an event, enabling durability and replay
2. **Four services work together** - Frontend (API), History (state), Matching (queues), Worker (background)
3. **Sharding enables scale** - Fixed shards, horizontal scaling via more hosts
4. **Sync matching = speed** - <10ms task dispatch by matching workers with waiting tasks
5. **Trade-offs are deliberate** - Storage for durability, complexity for scalability

---

## What's Next?

In [Blog Post 2](02-deep-dive-workflow-execution.md), we'll trace a complete workflow execution from start to finish, showing exactly how History Service manages state, how events are persisted, and how workers interact with the system.

**Topics Covered:**
- Step-by-step workflow execution flow
- Mutable State vs. History Events
- Transfer tasks and timer tasks
- How workers communicate with Matching
- Error handling and retries in action

---

## Further Reading

### Official Documentation
- [Temporal Concepts](https://docs.temporal.io/concepts) - User-facing concept docs
- [Architecture Overview](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/docs/architecture/README.md) - Internal architecture docs

### Related Blog Posts
- [Blog Post 3: Design Patterns](03-patterns-practices.md) - Deeper dive into architectural patterns
- [Blog Post 5: Performance](05-performance-analysis.md) - Benchmarks and optimization

### Research Papers
- [Event Sourcing Pattern](https://martinfowler.com/eaaDev/EventSourcing.html) - Martin Fowler (2005)
- [SWIM: Scalable Weakly-consistent Infection-style Process Group Membership Protocol](https://www.cs.cornell.edu/projects/Quicksilver/public_pdfs/SWIM.pdf) - Cornell (2002)

---

## Discussion Questions

For your team:
1. How would event sourcing benefit (or complicate) our current architecture?
2. What workflows in our system could benefit from Temporal's durability?
3. How does our current retry/timeout strategy compare to Temporal's approach?
4. What would our shard count need to be for our expected scale?

---

**Author's Note:** This analysis is based on commit `0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3`. The codebase evolves rapidly—check the latest main branch for updates.

**Feedback?** Found this helpful? Have questions? [Open a discussion](https://github.com/temporalio/temporal/discussions) or join the [Temporal Slack](https://t.mp/slack).
