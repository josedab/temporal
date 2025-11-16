# Deep Dive: The Lifecycle of a Workflow Execution

**Blog Series:** Part 2 of 7
**Commit SHA:** `0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3`
**Reading Time:** ~15 minutes
**Prerequisites:** [Part 1: Architecture Overview](01-architecture-overview.md)

## What You'll Learn

- Complete workflow execution lifecycle from start to completion
- How History Service manages workflow state with Mutable State
- The anatomy of Transfer Tasks and Timer Tasks
- How sync matching achieves <10ms task dispatch
- Error handling and retry mechanisms in action

---

## The Complete Workflow Journey

Let's trace a real workflow execution through Temporal's internals, examining each step in detail.

### Example Workflow

```go
// Simplified order processing workflow
func OrderWorkflow(ctx workflow.Context, orderID string) error {
    // Step 1: Charge payment
    var paymentID string
    err := workflow.ExecuteActivity(ctx, activities.ChargePayment, orderID).Get(ctx, &paymentID)
    if err != nil {
        return err
    }
    
    // Step 2: Wait 24 hours for fraud check
    err = workflow.Sleep(ctx, 24*time.Hour)
    if err != nil {
        return err
    }
    
    // Step 3: Ship order
    err = workflow.ExecuteActivity(ctx, activities.ShipOrder, orderID).Get(ctx, nil)
    if err != nil {
        // Compensate: refund payment
        _ = workflow.ExecuteActivity(ctx, activities.RefundPayment, paymentID).Get(ctx, nil)
        return err
    }
    
    return nil
}
```

---

## Phase 1: Workflow Start

### Client Initiates Workflow

```go
// User code
client.ExecuteWorkflow(ctx, client.StartWorkflowOptions{
    ID:        "order-12345",
    TaskQueue: "order-processing",
}, OrderWorkflow, "order-12345")
```

### Frontend Service Processing

**Location:** [`service/frontend/workflow_handler.go`](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/service/frontend/workflow_handler.go)

```
1. Validate request (workflow ID, namespace exists, etc.)
2. Generate RunID (UUID v4)
3. Calculate shard: hash(workflowID) % numShards
4. Route to History Service shard owner
```

**Shard Calculation:**
```go
// Simplified from common/persistence/shard.go
func WorkflowIDToShardID(workflowID string, numShards int32) int32 {
    hash := farm.Fingerprint64([]byte(workflowID))
    return int32(hash % uint64(numShards))
}
```

### History Service: Start Execution

**Location:** [`service/history/historyEngine.go`](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/service/history/historyEngine.go)

**Steps:**

```go
1. Acquire shard lock (ensures single owner)
2. Create Mutable State (in-memory workflow state)
3. Generate history events:
   - Event 1: WorkflowExecutionStarted
   - Event 2: WorkflowTaskScheduled
4. Create Transfer Task: "Add workflow task to matching"
5. Persist in SINGLE transaction:
   - History events → history_node table
   - Mutable state → executions table
   - Transfer task → transfer_tasks table
6. Release shard lock
```

**Critical: Atomicity**

```sql
BEGIN TRANSACTION;
  INSERT INTO history_node (shard_id, execution_id, event_id) VALUES (...);
  INSERT INTO executions (shard_id, workflow_id, mutable_state_blob) VALUES (...);
  INSERT INTO transfer_tasks (shard_id, task_id, task_type) VALUES (...);
COMMIT;
```

If ANY step fails, ENTIRE transaction rolls back. No partial state!

### Mutable State Structure

**Location:** [`service/history/workflow/mutable_state_impl.go`](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/service/history/workflow/mutable_state_impl.go)

```go
type MutableStateImpl struct {
    executionInfo  *persistencespb.WorkflowExecutionInfo
    executionState *persistencespb.WorkflowExecutionState
    
    // Pending work
    pendingActivityInfos     map[int64]*persistencespb.ActivityInfo
    pendingTimerInfos        map[string]*persistencespb.TimerInfo
    pendingChildExecutions   map[int64]*persistencespb.ChildExecutionInfo
    
    // Event buffering
    bufferedEvents           []*historypb.HistoryEvent
    updateInfos              map[string]*persistencespb.UpdateInfo
    
    // Tracking
    nextEventID              int64  // Next event to append
    lastWriteVersion         int64  // For replication
}
```

**Why Mutable State?**

Without it:
```
Get workflow status → Replay 10,000 events → Extract status → Return
(10+ seconds for large workflows!)
```

With Mutable State:
```
Get workflow status → Read cached state → Return
(<10ms!)
```

**Trade-off:** Memory overhead, cache invalidation complexity, but MASSIVE performance gain.

---

## Phase 2: Task Dispatch to Worker

### Transfer Task Processing

**Location:** [`service/history/queues/executable_task.go`](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/service/history/queues/executable_task.go)

History service runs **queue processors** that asynchronously process tasks:

```go
// Simplified queue processor loop
for {
    tasks := loadTasksFromDB(lastAckLevel, batchSize)
    for _, task := range tasks {
        switch task.Type {
        case TransferTaskTypeWorkflowTask:
            processWorkflowTask(task)
        case TransferTaskTypeActivityTask:
            processActivityTask(task)
        }
    }
    updateAckLevel(lastTaskID)
}
```

**Process Workflow Task:**
```go
func processWorkflowTask(task *TransferTask) {
    // Call Matching Service
    matchingClient.AddWorkflowTask(ctx, &matchingservice.AddWorkflowTaskRequest{
        NamespaceId: task.NamespaceID,
        Execution:   task.Execution,
        TaskQueue:   task.TaskQueue,
    })
    // Task now in Matching's queue
}
```

### Matching Service: The Magic of Sync Match

**Location:** [`service/matching/matcher.go`](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/service/matching/matcher.go)

**Scenario 1: Sync Match (Fast Path)**

```
Timeline:
T=0s:    Worker calls PollWorkflowTaskQueue (long-poll, 60s timeout)
T=0.001s: Worker is waiting...
T=0.005s: History calls AddWorkflowTask
T=0.006s: Matching IMMEDIATELY dispatches to waiting worker
T=0.007s: Worker receives task

Total latency: 7ms!
```

**Implementation:**
```go
type Matcher struct {
    fwdChan     chan *InternalTask      // Forward tasks to waiting workers
    waitingPollersCount atomic.Int32
}

func (m *Matcher) PollForTask(ctx context.Context) (*InternalTask, error) {
    m.waitingPollersCount.Inc()
    defer m.waitingPollersCount.Dec()
    
    select {
    case task := <-m.fwdChan:
        return task, nil  // Immediate match!
    case <-ctx.Done():
        return nil, ctx.Err()  // Timeout
    }
}

func (m *Matcher) Offer(task *InternalTask) bool {
    if m.waitingPollersCount.Load() > 0 {
        select {
        case m.fwdChan <- task:
            return true  // Sync matched!
        default:
            return false  // No waiting worker
        }
    }
    return false  // No pollers
}
```

**Scenario 2: Async Match (Slow Path)**

```
Timeline:
T=0s:    History calls AddWorkflowTask
T=0.001s: No waiting workers → persist task to database
T=5.000s: Worker calls PollWorkflowTaskQueue
T=5.050s: Load task from database
T=5.100s: Worker receives task

Total latency: 5.1 seconds (but still resilient!)
```

**Trade-off:** Sync match is 500-1000x faster, but requires workers to be pre-polling.

---

## Phase 3: Worker Executes Workflow Code

### Worker Receives Workflow Task

**SDK Code (temporalio/sdk-go):**
```go
// Worker polls Matching
task, err := client.PollWorkflowTaskQueue(ctx, &workflowservice.PollWorkflowTaskQueueRequest{
    Namespace: "production",
    TaskQueue: &taskqueuepb.TaskQueue{Name: "order-processing"},
})

// Task contains:
// - History events since last workflow task
// - Workflow execution context
// - Previous decisions/commands
```

### Worker Replays History

**Key Insight:** Workers DON'T have workflow state. They reconstruct it by replaying history events.

```go
// SDK internal replay logic
func (w *workflowExecutor) ProcessWorkflowTask(task *WorkflowTask) (*WorkflowTaskResult, error) {
    // Create new workflow state
    workflowState := newWorkflowState()
    
    // Replay ALL history events
    for _, event := range task.History.Events {
        switch event.Type {
        case WorkflowExecutionStarted:
            // Start workflow coroutine
            workflowState.start()
        case ActivityTaskScheduled:
            // Record that activity was scheduled
            workflowState.recordActivityScheduled(event.ActivityID)
        case ActivityTaskCompleted:
            // Resume workflow with activity result
            workflowState.resumeWithResult(event.ActivityID, event.Result)
        }
    }
    
    // Execute workflow code (may yield at ExecuteActivity calls)
    commands := workflowState.executeUntilBlocked()
    
    return &WorkflowTaskResult{Commands: commands}
}
```

### Worker Generates Commands

Our example workflow executes:
```go
err := workflow.ExecuteActivity(ctx, activities.ChargePayment, orderID).Get(ctx, &paymentID)
```

This YIELDS a command:
```go
commands = [
    {
        CommandType: ScheduleActivityTask,
        Attributes: {
            ActivityID:   "activity-1",
            ActivityType: "ChargePayment",
            Input:        marshaled("order-12345"),
            Timeout:      30s,
        },
    },
]
```

### Worker Completes Workflow Task

```go
client.RespondWorkflowTaskCompleted(ctx, &workflowservice.RespondWorkflowTaskCompletedRequest{
    TaskToken: task.TaskToken,
    Commands:  commands,
})
```

---

## Phase 4: Processing Commands

### History Service: Apply Commands

**Location:** [`service/history/workflow/workflow_task_state_machine.go`](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/service/history/workflow/workflow_task_state_machine.go)

```go
for _, command := range request.Commands {
    switch command.CommandType {
    case ScheduleActivityTask:
        // Generate events
        events = append(events,
            &historypb.HistoryEvent{
                EventType: WorkflowTaskCompleted,
                EventID:   nextEventID++,
            },
            &historypb.HistoryEvent{
                EventType: ActivityTaskScheduled,
                EventID:   nextEventID++,
                Attributes: &historypb.ActivityTaskScheduledEventAttributes{
                    ActivityID:   command.ActivityID,
                    ActivityType: command.ActivityType,
                },
            },
        )
        
        // Create transfer task
        transferTasks = append(transferTasks, &persistencespb.TransferTaskInfo{
            TaskType:   TransferTaskTypeActivityTask,
            TaskQueue:  command.TaskQueue,
            ActivityID: command.ActivityID,
        })
        
        // Update mutable state
        mutableState.AddActivityScheduled(command.ActivityID, ...)
    }
}

// Persist atomically
transaction {
    INSERT events INTO history_node
    UPDATE mutableState IN executions
    INSERT transferTasks INTO transfer_tasks
}
```

**Event IDs are Sequential:**
```
Event 1: WorkflowExecutionStarted
Event 2: WorkflowTaskScheduled
Event 3: WorkflowTaskStarted
Event 4: WorkflowTaskCompleted
Event 5: ActivityTaskScheduled  ← Just created
```

This sequentiality is CRITICAL for replay determinism.

---

## Phase 5: Activity Execution

### Activity Task Dispatch

Same process as workflow task:
1. Transfer task processor calls Matching.AddActivityTask()
2. Matching tries sync match with waiting worker
3. If no worker, persist to database
4. Worker polls and receives activity task

### Worker Executes Activity

**Unlike workflows, activities can have side effects:**

```go
func ChargePayment(ctx context.Context, orderID string) (string, error) {
    // Real I/O! Not deterministic!
    paymentID, err := stripeClient.Charge(ctx, &stripe.ChargeParams{
        Amount:      1000,
        Currency:    "usd",
        OrderID:     orderID,
    })
    if err != nil {
        // Temporal will retry this activity
        return "", err
    }
    
    // Heartbeat for long-running operations
    activity.RecordHeartbeat(ctx, "charged")
    
    return paymentID, nil
}
```

### Activity Completion

```go
client.RespondActivityTaskCompleted(ctx, &workflowservice.RespondActivityTaskCompletedRequest{
    TaskToken: task.TaskToken,
    Result:    marshaled(paymentID),
})
```

History Service:
```go
// Append events
events = [
    {EventType: ActivityTaskStarted, EventID: 6},
    {EventType: ActivityTaskCompleted, EventID: 7, Result: "pay_abc"},
]

// Schedule new workflow task (workflow needs to react to completion)
transferTasks = [
    {TaskType: TransferTaskTypeWorkflowTask},
]
```

---

## Phase 6: Timer Execution

Our workflow has a sleep:
```go
err = workflow.Sleep(ctx, 24*time.Hour)
```

### Worker Issues StartTimer Command

```go
commands = [
    {
        CommandType: StartTimer,
        Attributes: {
            TimerID:    "timer-1",
            Duration:   24h,
        },
    },
]
```

### History Service: Schedule Timer Task

```go
// Calculate fire time
fireTime := now.Add(24 * time.Hour)

// Create timer task
timerTask := &persistencespb.TimerTaskInfo{
    TaskType:       TimerTaskTypeWorkflowTimer,
    VisibilityTime: fireTime,  // When to fire
    TimerID:        "timer-1",
}

// Persist
transaction {
    INSERT event: TimerStarted
    UPDATE mutableState (add pending timer)
    INSERT timerTask INTO timer_tasks
}
```

### Timer Task Processor

**Location:** [`service/history/queues/timer_queue.go`](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/service/history/queues/timer_queue.go)

```go
// Runs every 100ms
for {
    now := time.Now()
    tasks := loadTimerTasks(visibilityTime <= now)
    
    for _, task := range tasks {
        switch task.TaskType {
        case TimerTaskTypeWorkflowTimer:
            fireTimer(task)
        case TimerTaskTypeActivityTimeout:
            timeoutActivity(task)
        case TimerTaskTypeWorkflowTimeout:
            timeoutWorkflow(task)
        }
    }
    
    sleep(100 * time.Millisecond)
}
```

**Fire Timer:**
```go
func fireTimer(task *TimerTask) {
    // Append event
    events = [{EventType: TimerFired, EventID: 8, TimerID: "timer-1"}]
    
    // Schedule workflow task
    transferTasks = [{TaskType: TransferTaskTypeWorkflowTask}]
    
    // Persist
    transaction { ... }
}
```

**24 hours later**, workflow resumes and continues to next step.

---

## Phase 7: Error Handling and Retries

### Activity Fails

Suppose `ShipOrder` activity fails:

```go
func ShipOrder(ctx context.Context, orderID string) error {
    err := warehouseAPI.Ship(orderID)
    if err != nil {
        return err  // Transient network error
    }
    return nil
}
```

Worker reports failure:
```go
client.RespondActivityTaskFailed(ctx, &workflowservice.RespondActivityTaskFailedRequest{
    TaskToken: task.TaskToken,
    Failure:   marshaled(err),
})
```

### History Service: Retry Logic

**Location:** [`service/history/workflow/activity_state_machine.go`](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/service/history/workflow/activity_state_machine.go)

```go
func (s *activityStateMachine) handleActivityFailed(failure *failurepb.Failure) {
    activityInfo := s.mutableState.GetActivityInfo(s.activityID)
    retryPolicy := activityInfo.RetryPolicy
    
    // Check if retryable
    if !isRetryable(failure) {
        // Non-retryable → fail to workflow
        s.failActivityToWorkflow(failure)
        return
    }
    
    // Check retry attempts
    if activityInfo.Attempt >= retryPolicy.MaximumAttempts {
        // Exhausted retries → fail to workflow
        s.failActivityToWorkflow(failure)
        return
    }
    
    // Calculate backoff
    backoff := calculateBackoff(
        activityInfo.Attempt,
        retryPolicy.InitialInterval,
        retryPolicy.BackoffCoefficient,
        retryPolicy.MaximumInterval,
    )
    
    // Schedule retry via timer task
    retryTime := now.Add(backoff)
    timerTask := &persistencespb.TimerTaskInfo{
        TaskType:       TimerTaskTypeActivityRetry,
        VisibilityTime: retryTime,
        ActivityID:     s.activityID,
    }
    
    // Persist
    events = [{EventType: ActivityTaskFailed, Attempt: 1}]
    transaction {
        INSERT events
        INSERT timerTask
        UPDATE mutableState (increment attempt)
    }
}
```

**Exponential Backoff:**
```
Attempt 1: Delay = 1s
Attempt 2: Delay = 2s (1s * 2.0)
Attempt 3: Delay = 4s (2s * 2.0)
Attempt 4: Delay = 8s (4s * 2.0)
...
Max:       Delay = 100s (capped)
```

After backoff timer fires, activity is retried automatically!

---

## Phase 8: Workflow Completion

Finally, workflow completes:
```go
return nil  // Success!
```

Worker sends:
```go
commands = [
    {
        CommandType: CompleteWorkflowExecution,
        Attributes: {
            Result: nil,  // No return value
        },
    },
]
```

History Service:
```go
// Final events
events = [
    {EventType: WorkflowTaskCompleted, EventID: 20},
    {EventType: WorkflowExecutionCompleted, EventID: 21},
]

// Update execution state
mutableState.ExecutionState.State = WorkflowExecutionStateCompleted

// Persist
transaction {
    INSERT events
    UPDATE mutableState
}

// Schedule archival (if configured)
if archivalEnabled {
    startArchivalWorkflow(workflowID, runID)
}
```

**Workflow is done!** History is preserved forever (or until archived).

---

## Key Data Structures

### History Event
```protobuf
message HistoryEvent {
    int64 event_id = 1;
    int64 timestamp = 2;
    EventType event_type = 3;
    int64 version = 4;  // For replication
    int64 task_id = 5;
    
    oneof attributes {
        WorkflowExecutionStartedEventAttributes workflow_execution_started = 10;
        ActivityTaskScheduledEventAttributes activity_task_scheduled = 11;
        // ... 50+ event types
    }
}
```

### Transfer Task
```protobuf
message TransferTaskInfo {
    string namespace_id = 1;
    string workflow_id = 2;
    string run_id = 3;
    int64 task_id = 4;
    TransferTaskType task_type = 5;
    string task_queue = 6;
    int64 schedule_id = 7;  // Event ID that scheduled this task
}
```

### Timer Task
```protobuf
message TimerTaskInfo {
    string namespace_id = 1;
    string workflow_id = 2;
    string run_id = 3;
    int64 task_id = 4;
    TimerTaskType task_type = 5;
    google.protobuf.Timestamp visibility_time = 6;  // When to fire
}
```

---

## Performance Characteristics

| Operation | Latency | Throughput |
|-----------|---------|------------|
| **Start Workflow** | 10-50ms | 1K-10K/sec (DB-dependent) |
| **Sync Match** | <10ms | 50K+ tasks/sec |
| **Async Match** | 100-500ms | 5K-20K tasks/sec |
| **Event Append** | 10-50ms | DB-dependent |
| **Workflow Replay** | 1-100ms | Depends on history size |

**Bottlenecks:**
1. Database write latency (events, mutable state)
2. Large history replay (10K+ events)
3. Hot shards (many workflows on one shard)

See [Blog Post 5: Performance Analysis](05-performance-analysis.md) for optimization strategies.

---

## Key Takeaways

1. **Event sourcing enables replay** - Every decision is recorded as an event
2. **Mutable State is a cache** - Performance optimization, not source of truth
3. **Transfer/Timer tasks decouple services** - History doesn't directly call Matching
4. **Sync matching is magic** - Pre-polling workers get <10ms task dispatch
5. **Retries are automatic** - Exponential backoff, configurable policies
6. **Atomicity is critical** - All state changes in single transaction

---

## What's Next?

[Blog Post 3: Design Patterns](03-patterns-practices.md) explores the architectural patterns in depth:
- Event sourcing vs. CQRS
- Transactional outbox pattern
- Sharding strategies
- Observability patterns

---

## Further Reading

- [History Service Architecture](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/docs/architecture/history-service.md)
- [Event Sourcing Pattern](https://martinfowler.com/eaaDev/EventSourcing.html) - Martin Fowler
- [Transactional Outbox Pattern](https://microservices.io/patterns/data/transactional-outbox.html)

---

**Questions?** [Join Temporal Slack](https://t.mp/slack) or [open a discussion](https://github.com/temporalio/temporal/discussions)
