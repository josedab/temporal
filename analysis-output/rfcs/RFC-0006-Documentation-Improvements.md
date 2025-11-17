# RFC-0006: Documentation Improvements

**Status:** Draft
**Effort:** 2-3 weeks
**Impact:** High (Long-term)
**Author:** Analysis Team
**Created:** 2025-11-16
**Commit SHA:** `0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3`

---

## Summary

Create comprehensive operator runbooks, troubleshooting guides, architecture deep-dives, and contributor onboarding documentation to reduce time-to-productivity for operators (from weeks to days) and contributors (from months to weeks). This addresses the current gap where architecture is well-documented but operational procedures and advanced features are under-documented.

---

## Motivation

### Current Documentation State

**Strengths ✅:**
- Good high-level architecture docs (`docs/architecture/README.md`)
- Clear setup instructions (`CONTRIBUTING.md`)
- Comprehensive code comments
- API documentation (generated from protobuf)

**Gaps ❌:**

#### 1. No Operator Runbooks

**Problem:** Operators don't have step-by-step guides for common operations.

**Examples of missing procedures:**
- How to upgrade from 1.24 → 1.25 with zero downtime
- What to do when History service is at 90% CPU
- How to recover from database corruption
- How to migrate from 512 shards to 4096 shards
- Emergency procedures for shard rebalancing

**Impact:**
```
Time to resolve production issues:
├─ Current: 45-60 minutes (trial and error, Slack questions)
└─ With runbooks: 10-15 minutes (follow documented procedure)

Operator onboarding time:
├─ Current: 3-4 weeks (learn by osmosis)
└─ With runbooks: 1 week (structured learning)
```

---

#### 2. Troubleshooting is Trial-and-Error

**Problem:** No systematic troubleshooting guide.

**Example scenario:**
```
Symptom: "Workflows are timing out"

Current process:
1. Check logs (which logs? what to grep for?)
2. Check metrics (which metrics? what's normal?)
3. Ask in #temporal-help (wait for response)
4. Try random things
5. Eventually find root cause (2-3 hours later)

Desired process:
1. Follow troubleshooting tree in docs
2. Identify root cause in 15 minutes
3. Apply documented fix
```

**Common issues without documentation:**
- High workflow task latency
- Sync match queue backlog
- Persistence layer timeouts
- Memory leaks in History service
- Elasticsearch query slowness
- Worker connection storms

---

#### 3. Advanced Features Under-Documented

**Problem:** Features exist but aren't well explained.

**Under-documented features:**
- **Nexus** - Cross-namespace orchestration (added 2024, sparse docs)
- **Worker Versioning** - Deployment safety (complex, examples lacking)
- **CHASM State Machines** - Advanced workflows (no tutorial)
- **Custom Persistence** - Implementing new databases (interface docs only)
- **Dynamic Config** - Runtime tuning (no comprehensive guide)
- **Archival** - Long-term storage (basic example only)

**Impact:**
- Features underutilized (users don't know they exist or how to use them)
- Misuse leads to production issues
- Community asks repetitive questions

---

#### 4. Contributor Ramp-Up Too Slow

**Problem:** New contributors take 2-3 months to be productive.

**Missing contributor docs:**
- Code walkthrough (where to start reading)
- Design patterns used (Fx, event sourcing, CQRS)
- Testing strategies
- How to add a new History service API
- How to add a new persistence store
- How to add metrics/tracing

**Current contributor experience:**
```
Week 1: Read code, confused
Week 2: Ask questions, start small PR
Week 3-4: PR feedback, learn more
Week 5-8: Second PR, starting to understand
Week 9+: Finally productive
```

**Goal: Reduce to 3-4 weeks** with structured onboarding docs.

---

### Quantitative Impact

| Metric | Current | Target | Improvement |
|--------|---------|--------|-------------|
| **MTTR (Mean Time to Resolution)** | 45-60 min | 10-15 min | **75% reduction** |
| **Operator Onboarding** | 3-4 weeks | 1 week | **75% reduction** |
| **Contributor Time to First Merged PR** | 4-6 weeks | 2 weeks | **66% reduction** |
| **Slack Support Questions** | ~50/week | ~20/week | **60% reduction** |
| **Documentation Satisfaction** | 3.2/5 (est.) | 4.5+/5 | **40% improvement** |

---

## Proposed Solution

### Documentation Architecture

```
docs/
├── architecture/          [EXISTS - Good]
│   ├── README.md
│   ├── history-service.md
│   └── ...
│
├── operator-runbooks/     [NEW]
│   ├── 00-index.md
│   ├── deployment/
│   │   ├── fresh-install.md
│   │   ├── kubernetes.md
│   │   ├── docker-compose.md
│   │   └── cloud-specific/
│   ├── operations/
│   │   ├── upgrades.md
│   │   ├── scaling.md
│   │   ├── backups.md
│   │   ├── monitoring.md
│   │   └── disaster-recovery.md
│   └── troubleshooting/
│       ├── workflows-timing-out.md
│       ├── high-latency.md
│       ├── database-issues.md
│       └── memory-leaks.md
│
├── advanced-features/     [NEW]
│   ├── nexus/
│   │   ├── overview.md
│   │   ├── tutorial.md
│   │   └── production.md
│   ├── worker-versioning/
│   ├── archival/
│   ├── custom-persistence/
│   └── dynamic-config/
│
├── contributors/          [ENHANCE EXISTING]
│   ├── 00-getting-started.md
│   ├── code-walkthrough.md
│   ├── architecture-deep-dive.md
│   ├── testing-guide.md
│   ├── how-to-add-api.md
│   └── design-patterns.md
│
└── troubleshooting/       [NEW]
    ├── 00-decision-tree.md
    ├── performance-tuning.md
    ├── common-errors.md
    └── debug-tools.md
```

---

## Detailed Design

### 1. Operator Runbooks

#### 1.1 Deployment Runbook

**File:** `docs/operator-runbooks/deployment/kubernetes.md`

**Content Structure:**
```markdown
# Deploying Temporal on Kubernetes

## Overview
- Architecture diagram
- Component requirements
- Resource recommendations

## Prerequisites
- Kubernetes 1.25+
- Database (PostgreSQL 12+, Cassandra 4.0+, or MySQL 8+)
- Optional: Elasticsearch 7.10+ for advanced visibility

## Step-by-Step Deployment

### Step 1: Database Setup

**PostgreSQL Example:**
```sql
CREATE DATABASE temporal;
CREATE DATABASE temporal_visibility;

CREATE USER temporal_user WITH PASSWORD 'secure-password';
GRANT ALL PRIVILEGES ON DATABASE temporal TO temporal_user;
GRANT ALL PRIVILEGES ON DATABASE temporal_visibility TO temporal_user;
```

**Run schema migration:**
```bash
temporal-sql-tool \
  --plugin postgres \
  --ep localhost \
  --port 5432 \
  --db temporal \
  --user temporal_user \
  --password secure-password \
  create
```

### Step 2: Configure Temporal

**Create ConfigMap:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: temporal-config
data:
  config.yaml: |
    persistence:
      defaultStore: default
      visibilityStore: visibility
      numHistoryShards: 512  # CRITICAL: Cannot change later
      datastores:
        default:
          sql:
            pluginName: "postgres"
            databaseName: "temporal"
            connectAddr: "postgres.default.svc.cluster.local:5432"
            connectProtocol: "tcp"
            user: "temporal_user"
            password: "secure-password"
        visibility:
          sql:
            pluginName: "postgres"
            databaseName: "temporal_visibility"
            connectAddr: "postgres.default.svc.cluster.local:5432"
            user: "temporal_user"
            password: "secure-password"
```

[...continues with full deployment steps]
```

**Key Sections:**
1. Prerequisites checklist
2. Database setup (all supported databases)
3. Kubernetes manifests (Helm charts)
4. Configuration examples
5. Verification steps
6. Common deployment issues

---

#### 1.2 Upgrade Runbook

**File:** `docs/operator-runbooks/operations/upgrades.md`

**Content:**
```markdown
# Zero-Downtime Upgrade Procedure

## Overview
Temporal supports rolling upgrades with zero downtime for minor versions.

## Pre-Upgrade Checklist

- [ ] Read release notes for breaking changes
- [ ] Backup database (see backup runbook)
- [ ] Test upgrade in staging environment
- [ ] Verify worker compatibility
- [ ] Schedule maintenance window (for major versions)
- [ ] Notify stakeholders

## Upgrade Process (Minor Versions: 1.24 → 1.25)

### Step 1: Upgrade Database Schema

```bash
# Download new schema files
wget https://github.com/temporalio/temporal/releases/download/v1.25.0/temporal-schema.tar.gz
tar -xzf temporal-schema.tar.gz

# Apply schema changes (dry run first)
temporal-sql-tool \
  --plugin postgres \
  --ep localhost \
  --db temporal \
  update \
  --schema-dir schema/postgresql/v12/temporal/versioned

# Verify no errors before proceeding
```

### Step 2: Rolling Service Upgrade

**Order matters:** Worker → Matching → History → Frontend

**Why this order?**
- Workers are stateless (safest to upgrade first)
- History depends on Matching
- Frontend depends on all other services

**For each service:**
```bash
# 1. Update image in deployment
kubectl set image deployment/temporal-history \
  temporal-history=temporalio/server:1.25.0

# 2. Watch rollout
kubectl rollout status deployment/temporal-history

# 3. Verify health
kubectl exec -it temporal-history-0 -- \
  curl http://localhost:7233/health

# 4. Check metrics for errors
# Look for: temporal_service_error_* metrics

# 5. Wait 5 minutes, monitor for issues

# 6. Proceed to next service
```

### Step 3: Post-Upgrade Verification

```bash
# Run smoke tests
temporal workflow start \
  --type TestWorkflow \
  --task-queue test-queue \
  --workflow-id upgrade-test-$(date +%s)

# Check workflow completes
temporal workflow show --workflow-id upgrade-test-*

# Verify metrics baseline
# - Workflow start rate should be normal
# - Task latency should be normal
# - Error rate should be <0.1%
```

### Rollback Procedure

**If issues detected within 1 hour:**
```bash
# 1. Rollback deployment
kubectl rollout undo deployment/temporal-history

# 2. Verify rollback successful
kubectl rollout status deployment/temporal-history

# 3. Check metrics return to normal

# 4. Investigate root cause before retry
```

## Major Version Upgrades (1.x → 2.x)

Major versions may require downtime. See specific release notes.

[...continues with major version specifics]
```

---

#### 1.3 Disaster Recovery Runbook

**File:** `docs/operator-runbooks/operations/disaster-recovery.md`

**Content:**
```markdown
# Disaster Recovery Procedures

## Scenario 1: Database Corruption

**Symptoms:**
- Workflows failing with "corrupted mutable state"
- History service crashes on specific workflows

**Recovery:**
```bash
# 1. Identify affected workflows
grep "corrupted mutable state" /var/log/temporal/history.log | \
  awk '{print $5}' | sort -u > affected-workflows.txt

# 2. For each workflow, attempt reset
while read workflow_id; do
  temporal workflow reset \
    --workflow-id "$workflow_id" \
    --reason "database corruption recovery" \
    --event-id 1
done < affected-workflows.txt

# 3. If reset fails, terminate and restart
while read workflow_id; do
  temporal workflow terminate \
    --workflow-id "$workflow_id" \
    --reason "unrecoverable corruption"

  # Restart workflow if idempotent
  temporal workflow start \
    --workflow-id "${workflow_id}-recovered" \
    --type OriginalWorkflowType
done < affected-workflows.txt
```

## Scenario 2: Complete Database Loss

**Prerequisites:**
- Recent backup exists (see backup runbook)
- RTO (Recovery Time Objective): 4 hours
- RPO (Recovery Point Objective): 1 hour

**Recovery Steps:**

```bash
# 1. Provision new database
# [database-specific steps]

# 2. Restore from backup
pg_restore -d temporal /backups/temporal-$(date -d yesterday +%Y%m%d).dump

# 3. Update Temporal configuration
kubectl edit configmap temporal-config
# Update database connection strings

# 4. Restart all services
kubectl rollout restart deployment/temporal-*

# 5. Verify recovery
temporal operator namespace list
temporal workflow count --query "StartTime > '$(date -d '2 hours ago' --iso-8601)'"

# 6. Reconcile workflows since backup
# [specific to your workflow patterns]
```

[...continues with more scenarios]
```

---

### 2. Troubleshooting Guide

#### 2.1 Troubleshooting Decision Tree

**File:** `docs/troubleshooting/00-decision-tree.md`

**Content:**
```markdown
# Temporal Troubleshooting Decision Tree

## Start Here

**Symptom:** Workflows are slow/timing out

### Step 1: Identify the Component

Check which service is slow:

```bash
# Check service latency metrics
curl -s http://temporal-frontend:9090/metrics | \
  grep temporal_service_latency_bucket

# Expected values:
# - Frontend: p99 < 100ms
# - History: p99 < 200ms
# - Matching: p99 < 50ms
```

**If Frontend is slow** → See [Frontend Issues](#frontend-issues)
**If History is slow** → See [History Issues](#history-issues)
**If Matching is slow** → See [Matching Issues](#matching-issues)
**If Database is slow** → See [Database Issues](#database-issues)

---

## Frontend Issues

### High Frontend Latency (p99 > 100ms)

**Possible causes:**
1. **Database connection pool exhausted**
2. **Rate limiting kicking in**
3. **Namespace cache miss**

**Diagnosis:**
```bash
# Check connection pool
curl http://frontend:9090/metrics | grep sql_connection_

# Expected: sql_connection_active < sql_connection_max * 0.8

# Check rate limiter
curl http://frontend:9090/metrics | grep ratelimit_

# Expected: rate_limited_requests < 1% of total_requests
```

**Fix for connection pool exhaustion:**
```yaml
# Update dynamic config
persistence:
  defaultStore:
    sql:
      maxConns: 100  # Increase from default 20
      maxIdleConns: 50
```

[...continues with detailed troubleshooting for each issue]

---

## History Issues

### High Memory Usage (>80% of pod limit)

**Symptoms:**
- History pods OOMKilled
- Increasing memory usage over time
- Slow workflow task processing

**Diagnosis:**
```bash
# Check mutable state cache size
curl http://history:9090/metrics | \
  grep temporal_mutable_state_cache_

# Check for workflow leaks
temporal workflow list \
  --query 'ExecutionStatus = "Running" AND StartTime < "7 days ago"' \
  --limit 100
```

**Root causes:**

1. **Too many active workflows per shard**
   ```
   Calculation:
   Workflows per shard = Total active workflows / Shard count

   Healthy: < 10,000 per shard
   Warning: 10,000 - 50,000 per shard
   Critical: > 50,000 per shard
   ```

   **Fix:** Scale horizontally (add History pods)
   ```bash
   kubectl scale deployment/temporal-history --replicas=10
   ```

2. **Workflow history too large (>50K events)**
   ```bash
   # Find large workflows
   SELECT workflow_id, execution_run_id,
          jsonb_array_length(history_node) as event_count
   FROM executions
   WHERE jsonb_array_length(history_node) > 50000
   ORDER BY event_count DESC
   LIMIT 10;
   ```

   **Fix:** Use continue-as-new
   ```go
   if workflow.GetInfo(ctx).GetCurrentHistoryLength() > 50000 {
       return workflow.NewContinueAsNewError(ctx, WorkflowFunc, args...)
   }
   ```

[...continues with more issues and fixes]
```

---

### 3. Advanced Feature Guides

#### 3.1 Nexus Deep Dive

**File:** `docs/advanced-features/nexus/tutorial.md`

**Content:**
```markdown
# Nexus: Cross-Namespace Orchestration

## What is Nexus?

Nexus enables workflows in one namespace to invoke workflows in another namespace (or even another cluster) without tight coupling.

**Use Cases:**
- Multi-tenant SaaS (isolate customer workflows)
- Microservices orchestration (each service has its own namespace)
- Cross-team collaboration (marketing team invokes billing team workflows)

## Architecture

```
┌─────────────────┐         ┌─────────────────┐
│  Namespace A    │         │  Namespace B    │
│                 │         │                 │
│  ┌───────────┐  │ Nexus   │  ┌───────────┐  │
│  │ Workflow1 ├──┼────────▶│  │ Workflow2 │  │
│  └───────────┘  │ Call    │  └───────────┘  │
│                 │         │                 │
└─────────────────┘         └─────────────────┘
```

## Step-by-Step Tutorial

### Step 1: Define Nexus Endpoint

```go
// File: billing-service/nexus.go
package billing

import (
    "go.temporal.io/sdk/nexus"
)

// BillingNexusService exposes billing operations
type BillingNexusService struct{}

func (s *BillingNexusService) ChargeCustomer(
    ctx context.Context,
    input ChargeInput,
) (ChargeResult, error) {
    // Start workflow in billing namespace
    workflowID := fmt.Sprintf("charge-%s-%d", input.CustomerID, time.Now().Unix())

    run, err := s.client.ExecuteWorkflow(ctx,
        client.StartWorkflowOptions{
            ID: workflowID,
            TaskQueue: "billing-queue",
        },
        ChargeWorkflow,
        input,
    )

    if err != nil {
        return ChargeResult{}, err
    }

    // Wait for result
    var result ChargeResult
    err = run.Get(ctx, &result)
    return result, err
}
```

### Step 2: Register Nexus Endpoint

```bash
# Register endpoint with Temporal
temporal operator nexus endpoint create \
  --name billing-service \
  --target-namespace billing-prod \
  --description "Billing operations"
```

### Step 3: Invoke from Another Namespace

```go
// File: order-service/workflows/order.go
package workflows

import (
    "go.temporal.io/sdk/workflow"
    "go.temporal.io/sdk/nexus"
)

func OrderWorkflow(ctx workflow.Context, order Order) error {
    // Invoke billing service via Nexus
    nexusClient := nexus.NewClient("billing-service")

    var chargeResult billing.ChargeResult
    err := nexusClient.ExecuteOperation(
        ctx,
        "ChargeCustomer",
        billing.ChargeInput{
            CustomerID: order.CustomerID,
            Amount:     order.Total,
        },
        &chargeResult,
    )

    if err != nil {
        return fmt.Errorf("charge failed: %w", err)
    }

    // Continue with order fulfillment
    return nil
}
```

### Step 4: Testing

```bash
# Start order workflow (in order-service namespace)
temporal workflow start \
  --namespace order-prod \
  --task-queue order-queue \
  --type OrderWorkflow \
  --workflow-id order-12345 \
  --input '{"customer_id": "cust-123", "total": 99.99}'

# Verify it invoked billing workflow (in billing-service namespace)
temporal workflow list \
  --namespace billing-prod \
  --query 'WorkflowType = "ChargeWorkflow"'
```

## Production Considerations

### 1. Timeouts

```go
nexusClient.ExecuteOperation(
    ctx,
    "ChargeCustomer",
    input,
    &result,
    nexus.OperationOptions{
        ScheduleToCloseTimeout: 30 * time.Second,
        // If billing service is slow, fail fast
    },
)
```

### 2. Error Handling

```go
err := nexusClient.ExecuteOperation(ctx, "ChargeCustomer", input, &result)
if err != nil {
    var nexusErr *nexus.Error
    if errors.As(err, &nexusErr) {
        switch nexusErr.Type {
        case nexus.HandlerErrorType:
            // Billing service rejected (e.g., invalid card)
            return workflow.NewApplicationError("payment declined", "PaymentDeclined")
        case nexus.TimeoutErrorType:
            // Billing service too slow
            return workflow.NewApplicationError("payment timeout", "PaymentTimeout")
        default:
            // Unexpected error
            return err
        }
    }
    return err
}
```

### 3. Monitoring

```bash
# Metrics to monitor
temporal_nexus_request_count           # Request rate
temporal_nexus_request_latency_bucket  # Latency
temporal_nexus_error_count             # Error rate
```

[...continues with more advanced topics]
```

---

### 4. Contributor Guides

#### 4.1 Code Walkthrough

**File:** `docs/contributors/code-walkthrough.md`

**Content:**
```markdown
# Temporal Server Code Walkthrough

## New Contributor Journey

**Goal:** Understand enough to make your first meaningful contribution.

**Estimated Time:** 4-8 hours of reading + experimentation

---

## Journey Map

```
Start → Core Concepts → Follow a Workflow → Make a Change → PR
  ↓         ↓              ↓                    ↓           ↓
 30min    1 hour        2-3 hours           2-3 hours    1 hour
```

---

## Part 1: Core Concepts (30 minutes)

Before diving into code, understand these concepts:

### Event Sourcing
**Read:** `docs/architecture/event-sourcing.md`

**Key insight:** Every workflow state change is an event. Mutable state is derived.

**Try it:**
```bash
# Start a workflow
temporal workflow start --type TestWorkflow --task-queue test --workflow-id test-1

# View history events
temporal workflow show --workflow-id test-1

# Notice: each step is an event (WorkflowExecutionStarted, ActivityTaskScheduled, etc.)
```

### Services Architecture
**Read:** `docs/architecture/README.md`

**Key files:**
- `service/frontend/handler.go` - API entry point
- `service/history/historyEngine.go` - Workflow state machine
- `service/matching/matchingEngine.go` - Task queue routing
- `service/worker/worker.go` - Background jobs

---

## Part 2: Follow a Workflow Execution (2-3 hours)

**Goal:** Trace a workflow from start to completion through the code.

### Step 1: Start Workflow (Frontend Service)

**Entry point:** `service/frontend/workflow_handler.go`

```go
// Line 245
func (wh *WorkflowHandler) StartWorkflowExecution(
    ctx context.Context,
    request *workflowservice.StartWorkflowExecutionRequest,
) (*workflowservice.StartWorkflowExecutionResponse, error) {

    // 1. Validate request
    if err := wh.validateStartWorkflowRequest(request); err != nil {
        return nil, err
    }

    // 2. Get namespace info
    namespace, err := wh.namespaceRegistry.GetNamespace(
        namespace.Name(request.GetNamespace()),
    )

    // 3. Forward to History service
    response, err := wh.historyClient.StartWorkflowExecution(ctx, &historyservice.StartWorkflowExecutionRequest{
        NamespaceId: namespace.ID().String(),
        StartRequest: request,
    })

    return response, err
}
```

**What happens:**
1. Frontend validates request
2. Looks up namespace
3. Forwards to History service (via gRPC)

**Debug it:**
```bash
# Add breakpoint in your IDE
# Or add log statement:
log.Info("StartWorkflowExecution called", tag.WorkflowID(request.WorkflowId))

# Rebuild and run
make bins
./temporal-server start
```

### Step 2: History Service Creates Workflow

**File:** `service/history/historyEngine.go`

```go
// Line ~680
func (e *historyEngineImpl) StartWorkflowExecution(
    ctx context.Context,
    request *historyservice.StartWorkflowExecutionRequest,
) (*historyservice.StartWorkflowExecutionResponse, error) {

    // 1. Determine shard
    shardID := e.shardController.ShardID(
        namespace.ID(request.NamespaceId),
        request.StartRequest.WorkflowId,
    )

    // 2. Get shard context
    shardContext, err := e.shardController.GetShardByID(shardID)

    // 3. Create mutable state
    mutableState := workflow.NewMutableState(...)

    // 4. Add WorkflowExecutionStarted event
    _, err = mutableState.AddWorkflowExecutionStartedEvent(
        request.StartRequest,
    )

    // 5. Create workflow task
    _, err = mutableState.AddWorkflowTaskScheduledEvent(...)

    // 6. Persist to database + create transfer task
    err = shardContext.CreateWorkflowExecution(ctx, &persistence.CreateWorkflowExecutionRequest{
        NewWorkflowSnapshot: mutableState.CloseTransactionAsSnapshot(),
    })

    return response, err
}
```

**Key concepts:**
- **Shard**: Workflow hashed to shard (consistent hashing)
- **Mutable State**: In-memory workflow state
- **Events**: WorkflowExecutionStarted, WorkflowTaskScheduled
- **Transfer Task**: Async task to notify Matching service

**Trace it:**
```bash
# Add logs at each step
# Rebuild and watch logs
./temporal-server start | grep "workflow-id=test-1"
```

### Step 3: Transfer Task Processor Dispatches Task

**File:** `service/history/queues/transfer_queue_active_task_executor.go`

```go
// Line ~150
func (t *transferQueueActiveTaskExecutor) executeWorkflowTaskScheduled(
    ctx context.Context,
    task *tasks.WorkflowTask,
) error {
    // 1. Load mutable state
    mutableState, err := t.shardContext.GetWorkflowExecution(ctx, task.WorkflowKey)

    // 2. Get workflow task info
    workflowTask := mutableState.GetWorkflowTaskInfo()

    // 3. Send to Matching service
    _, err = t.matchingClient.AddWorkflowTask(ctx, &matchingservice.AddWorkflowTaskRequest{
        NamespaceId: task.NamespaceID,
        Execution:   task.WorkflowID,
        TaskQueue:   workflowTask.TaskQueue,
        ScheduledEventId: workflowTask.ScheduledEventID,
    })

    return err
}
```

**What happens:**
- Transfer task processor runs async
- Picks up workflow task from database
- Sends to Matching service

### Step 4: Worker Polls and Executes

**File:** `service/matching/matchingEngine.go`

```go
// Line ~420
func (e *matchingEngineImpl) PollWorkflowTaskQueue(
    ctx context.Context,
    request *matchingservice.PollWorkflowTaskQueueRequest,
) (*matchingservice.PollWorkflowTaskQueueResponse, error) {

    // 1. Get task queue
    taskQueue, err := e.getTaskQueue(request.TaskQueue)

    // 2. Wait for task (long poll)
    task, err := taskQueue.PollTask(ctx)

    // 3. Return task to worker
    return &matchingservice.PollWorkflowTaskQueueResponse{
        Task: task,
    }, nil
}
```

**Worker then:**
1. Executes workflow code
2. Returns commands (ScheduleActivityTask, CompleteWorkflowExecution, etc.)
3. History service processes commands

---

## Part 3: Make Your First Change (2-3 hours)

**Good first issues:**
- Add a new metric
- Improve error message
- Add test case
- Fix documentation typo

**Example: Add a metric**

### 1. Find where to add metric

```go
// File: service/history/historyEngine.go
func (e *historyEngineImpl) StartWorkflowExecution(...) {

    // Add metric here
    e.metricsHandler.Counter("workflow_started").Inc(1)

    // Rest of function
}
```

### 2. Define metric

```go
// File: common/metrics/enum.go
const (
    WorkflowStarted = "workflow_started"
)
```

### 3. Test locally

```bash
make bins
./temporal-server start

# Start workflow
temporal workflow start --type TestWorkflow --task-queue test --workflow-id test-1

# Check metrics
curl http://localhost:9090/metrics | grep workflow_started
```

### 4. Add test

```go
// File: service/history/historyEngine_test.go
func (s *engineSuite) TestStartWorkflowExecution_EmitsMetric() {
    s.mockMetricsHandler.EXPECT().Counter("workflow_started").Return(s.mockCounter)
    s.mockCounter.EXPECT().Inc(1)

    _, err := s.engine.StartWorkflowExecution(...)
    s.NoError(err)
}
```

### 5. Submit PR

```bash
git checkout -b add-workflow-started-metric
git add .
git commit -m "Add workflow_started metric"
git push origin add-workflow-started-metric

# Create PR on GitHub
```

---

## Part 4: Deeper Dives (optional)

Once comfortable, explore:

### Persistence Layer
**Files:**
- `common/persistence/client/factory.go` - Persistence abstraction
- `common/persistence/sql/sqlplugin/postgresql/` - PostgreSQL implementation

### Event Sourcing Internals
**Files:**
- `service/history/workflow/mutable_state_impl.go` - State machine
- `service/history/workflow/history_builder.go` - Event builder

### Task Queue Mechanics
**Files:**
- `service/matching/physical_task_queue_manager.go` - Queue management
- `service/matching/task_writer.go` - Task persistence

---

## Resources

- **Architecture docs:** `docs/architecture/`
- **ADRs (Architecture Decision Records):** Coming soon
- **Design patterns:** See [design-patterns.md](design-patterns.md)
- **Slack:** #temporal-contributors

---

**Next:** [Testing Guide](testing-guide.md)
```

---

## Implementation Plan

### Week 1: Operator Runbooks

**Day 1: Deployment Runbooks**
```bash
# Create directory structure
mkdir -p docs/operator-runbooks/deployment
mkdir -p docs/operator-runbooks/operations

# Write runbooks
docs/operator-runbooks/deployment/
├── fresh-install.md         (800 lines)
├── kubernetes.md            (1,200 lines)
├── docker-compose.md        (600 lines)
└── cloud-specific/
    ├── aws.md
    ├── gcp.md
    └── azure.md
```

**Deliverables:**
- ✅ 5 deployment runbooks
- ✅ Tested on fresh Kubernetes cluster
- ✅ Peer-reviewed by SRE team

**Day 2-3: Operations Runbooks**
```bash
docs/operator-runbooks/operations/
├── upgrades.md              (1,500 lines)
├── scaling.md               (800 lines)
├── backups.md               (600 lines)
├── monitoring.md            (1,000 lines)
└── disaster-recovery.md     (1,200 lines)
```

**Deliverables:**
- ✅ 5 operational runbooks
- ✅ Verified procedures in staging
- ✅ Runbook automation scripts (where applicable)

**Day 4-5: Troubleshooting Runbooks**
```bash
docs/operator-runbooks/troubleshooting/
├── 00-decision-tree.md      (1,000 lines)
├── workflows-timing-out.md  (600 lines)
├── high-latency.md          (800 lines)
├── database-issues.md       (700 lines)
├── memory-leaks.md          (500 lines)
└── common-errors.md         (800 lines)
```

**Deliverables:**
- ✅ Troubleshooting decision tree
- ✅ 6 issue-specific runbooks
- ✅ Metric/log queries for diagnosis

---

### Week 2: Advanced Features & Contributors

**Day 1-2: Advanced Feature Guides**
```bash
docs/advanced-features/
├── nexus/
│   ├── overview.md          (400 lines)
│   ├── tutorial.md          (1,200 lines)
│   └── production.md        (600 lines)
├── worker-versioning/
│   ├── concepts.md          (500 lines)
│   ├── tutorial.md          (1,000 lines)
│   └── migration.md         (700 lines)
├── archival/
│   └── complete-guide.md    (800 lines)
├── custom-persistence/
│   └── implementing.md      (1,500 lines)
└── dynamic-config/
    └── reference.md         (1,000 lines)
```

**Day 3-5: Contributor Guides**
```bash
docs/contributors/
├── 00-getting-started.md    (600 lines) [ENHANCE]
├── code-walkthrough.md      (2,000 lines) [NEW]
├── architecture-deep-dive.md (1,500 lines) [NEW]
├── testing-guide.md         (1,200 lines) [ENHANCE]
├── how-to-add-api.md        (1,000 lines) [NEW]
└── design-patterns.md       (1,000 lines) [NEW]
```

**Deliverables:**
- ✅ 5 advanced feature guides with working code examples
- ✅ Comprehensive contributor onboarding path
- ✅ Code walkthrough tested with new contributor

---

### Week 3: Review, Polish, and Publishing

**Day 1-2: Internal Review**
- Submit all docs for review (10 reviewers)
- Incorporate feedback
- Technical accuracy verification

**Day 3: External Beta Review**
- Share with 5 community members
- Gather feedback on clarity
- Identify missing topics

**Day 4: Publishing**
- Integrate into docs.temporal.io
- Update navigation
- Create announcement blog post
- Update README with links

**Day 5: Metrics Setup**
- Setup doc analytics
- Create feedback forms
- Monitor Slack questions (should decrease)

---

## Success Metrics

### Quantitative Metrics

| Metric | Baseline | 1 Month | 3 Months | 6 Months |
|--------|----------|---------|----------|----------|
| **MTTR** | 45-60min | 30min | 15min | 10min |
| **Operator Onboarding** | 3-4 weeks | 2 weeks | 1 week | 1 week |
| **Contributor Time to First PR** | 4-6 weeks | 3 weeks | 2 weeks | 2 weeks |
| **Slack Support Questions** | ~50/week | ~35/week | ~20/week | ~15/week |
| **Doc Satisfaction** | 3.2/5 | 3.8/5 | 4.2/5 | 4.5/5 |
| **Doc Page Views** | - | 1,000/month | 5,000/month | 10,000/month |

### Qualitative Metrics

**Survey Questions (1-5 scale):**
1. "Documentation helped me solve my problem quickly"
2. "Examples were relevant to my use case"
3. "I would recommend Temporal based on documentation quality"

**Success Criteria:**
- ✅ All questions average >4.0 by Month 3
- ✅ No critical documentation gaps reported
- ✅ Positive feedback from 80%+ of surveyors

---

## Risks and Mitigations

### Risk 1: Documentation Becomes Outdated

**Likelihood:** High
**Impact:** Medium

**Mitigation:**
```yaml
# CI check for broken links
.github/workflows/docs-check.yml:
  - name: Check documentation links
    run: |
      find docs/ -name '*.md' -exec \
        markdown-link-check --config .markdown-link-check.json {} \;

# Quarterly documentation review
# Add to CODEOWNERS
/docs/ @temporal-docs-team

# Version documentation with code
docs/
  v1.24/
  v1.25/
  latest/ (symlink to current)
```

### Risk 2: Contributors Don't Use New Guides

**Likelihood:** Medium
**Impact:** Medium

**Mitigation:**
- Link from CONTRIBUTING.md
- Mention in PR template
- Onboarding buddy program references guides
- Measure usage via analytics

### Risk 3: Runbooks Don't Match Reality

**Likelihood:** Medium
**Impact:** High

**Mitigation:**
- Test all procedures in staging quarterly
- Runbook validation checklist in PR template
- Incident retrospectives update runbooks
- Community feedback loop

---

## Alternatives Considered

### Alternative 1: Use External Documentation Platform

**Platforms:** GitBook, ReadTheDocs, Docusaurus

**Pros:**
- Professional appearance
- Better search
- Versioning support

**Cons:**
- Another tool to maintain
- Migration effort
- Potential vendor lock-in

**Decision:** Keep docs in repo for now, consider Docusaurus for docs.temporal.io upgrade later

---

### Alternative 2: Video Tutorials Instead of Written Docs

**Pros:**
- More engaging
- Easier to follow along

**Cons:**
- Expensive to produce
- Hard to keep updated
- Not searchable
- Accessibility issues

**Decision:** Written docs primary, videos supplement (add in Phase 2)

---

### Alternative 3: AI Chatbot for Documentation

**Pros:**
- Interactive help
- Can answer specific questions

**Cons:**
- Hallucination risk
- Requires training data (written docs first!)
- Not a replacement for structured docs

**Decision:** Interesting for future, not a substitute for foundational docs

---

## Migration Strategy

### Phase 1: Create Core Runbooks (Week 1)

Start with most requested documentation:
1. Kubernetes deployment
2. Upgrade procedure
3. Troubleshooting decision tree

**Announce:** "New operator runbooks available at docs/operator-runbooks"

### Phase 2: Advanced Features (Week 2)

Fill knowledge gaps for new features:
1. Nexus tutorial
2. Worker versioning guide
3. Custom persistence implementation

**Announce:** "Advanced feature guides now available"

### Phase 3: Contributor Onboarding (Week 2-3)

Improve new contributor experience:
1. Code walkthrough
2. Architecture deep-dive
3. Testing guide

**Announce:** "New contributor onboarding path - reduce ramp-up time by 50%"

### Phase 4: Promotion (Week 3+)

- Blog post: "Temporal Documentation Overhaul"
- Slack announcement
- Twitter/social media
- Include in release notes

---

## Open Questions

1. **Q:** Should runbooks live in main repo or separate docs repo?
   **A:** Main repo (docs/ directory) for version alignment, publish to docs.temporal.io

2. **Q:** How to handle cloud-specific (AWS/GCP/Azure) documentation?
   **A:** Create cloud-specific subdirectories, maintain separately, community contributions welcome

3. **Q:** Video tutorials?
   **A:** Phase 2, after written docs are stable

4. **Q:** Translations?
   **A:** English first, community translations in Phase 2 if demand exists

---

## Related Work

- **RFC-0003: Developer Experience** - Local setup improvements complement documentation
- **RFC-0005: Testing Infrastructure** - Testing guide is part of contributor docs
- **Blog Series** - Blog posts serve as promotional content for new docs

---

## References

### Documentation Best Practices
- [Divio Documentation System](https://documentation.divio.com/) - Tutorials, how-tos, reference, explanation
- [Google Developer Documentation Style Guide](https://developers.google.com/style)
- [Write the Docs](https://www.writethedocs.org/guide/) - Community resources

### Example Runbooks
- [Kubernetes Runbooks](https://github.com/kubernetes/website/tree/main/content/en/docs/tasks)
- [GitLab Runbooks](https://gitlab.com/gitlab-com/runbooks)
- [Elasticsearch Operations Guide](https://www.elastic.co/guide/en/elasticsearch/reference/current/setup.html)

---

**Next Steps:**
1. Review and approve RFC
2. Assign documentation owners
3. Begin Week 1 runbook creation
4. Set up doc analytics

**Status:** Ready for review
**Estimated Completion:** 3 weeks from approval
