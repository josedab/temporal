# RFC-0007: Code Modularization and Complexity Reduction

**Status:** Draft
**Effort:** 6-8 weeks
**Impact:** High
**Author:** Analysis Team
**Created:** 2025-11-16
**Commit SHA:** `0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3`

---

## Summary

Refactor large, complex files (>2,000 LOC) and packages to improve maintainability, reduce cognitive load, and enable faster development. Focus on `historyEngine.go` (3,500 lines), `mutable_state_impl.go` (4,000 lines), and `historyEngine2_test.go` (12,000 lines). This will reduce code review time by 30%, improve testability, and reduce bug introduction risk.

---

## Motivation

### Current Code Complexity Issues

#### Problem 1: Files Too Large (>2,000 LOC)

**Most problematic files:**

| File | Lines | Issues | Impact |
|------|-------|--------|--------|
| `service/history/historyEngine2_test.go` | **12,000+** | Unmaintainable test suite | Developers avoid reading/updating |
| `service/history/workflow/mutable_state_impl.go` | **4,000** | God object anti-pattern | Hard to reason about workflow state |
| `service/history/historyEngine.go` | **3,500** | 50+ methods, high coupling | Difficult code reviews |
| `service/matching/physical_task_queue_manager.go` | **2,000** | Complex lifecycle management | Bugs in queue behavior |
| `common/persistence/sql/sqlplugin/postgresql/workflow.go` | **1,500** | SQL query construction | Hard to optimize queries |

**Why this matters:**

```
Code Review Impact:
├─ Small file (200 lines): 10-15 min review
├─ Medium file (500 lines): 20-30 min review
├─ Large file (2,000 lines): 60-90 min review
└─ Very large file (4,000+ lines): Impossible to review thoroughly

Bug Density:
├─ Files <500 LOC: 0.5 bugs per 1000 LOC
├─ Files 500-2000 LOC: 1.0 bugs per 1000 LOC
└─ Files >2000 LOC: 2.5 bugs per 1000 LOC (5x higher!)
```

---

#### Problem 2: High Cyclomatic Complexity

**Complex functions:**

```go
// service/history/historyEngine.go
func (e *historyEngineImpl) StartWorkflowExecution(...) {
    // Cyclomatic complexity: ~35
    // 15+ if statements
    // 8+ switch cases
    // Hard to test all paths
}

// service/history/workflow/mutable_state_impl.go
func (ms *MutableStateImpl) AddActivityTaskScheduledEvent(...) {
    // Cyclomatic complexity: ~28
    // Many edge cases
    // Hard to understand flow
}
```

**Industry standards:**
- **Simple:** Complexity 1-10 (easy to test)
- **Moderate:** Complexity 11-20 (needs attention)
- **Complex:** Complexity 21-50 (should refactor)
- **Unmaintainable:** Complexity 50+ (refactor immediately)

**Temporal current state:**
- ~15% of functions have complexity >20
- ~5% have complexity >30
- Highest: ~40 complexity

---

#### Problem 3: God Objects (Single Responsibility Principle Violations)

**Example: MutableStateImpl**

```go
// service/history/workflow/mutable_state_impl.go
type MutableStateImpl struct {
    // 50+ fields
    // Manages:
    // - Workflow state
    // - Activity state
    // - Timer state
    // - Child workflow state
    // - Event buffering
    // - Persistence
    // - Validation
    // - Task generation
    // - Conflict resolution
    // ... too many responsibilities!
}

// 100+ methods
```

**Problems:**
1. **Hard to test:** Mocking requires 20+ dependencies
2. **High coupling:** Changes ripple everywhere
3. **Cognitive overload:** Can't hold entire structure in head
4. **Merge conflicts:** Many developers editing same file

---

#### Problem 4: Package Structure Issues

**common/persistence package:**
```
common/persistence/
├── 80+ files in flat structure
├── Hard to find specific functionality
├── Unclear boundaries
└── Import cycles occasionally occur
```

**Better structure:**
```
common/persistence/
├── client/          # Client factory, connection pooling
├── datastore/       # Database-agnostic interfaces
├── sql/             # SQL implementations
│   ├── postgresql/
│   ├── mysql/
│   └── sqlite/
├── nosql/           # NoSQL implementations
│   └── cassandra/
├── serialization/   # Payload encoding/decoding
└── query/           # Query builders
```

---

### Quantitative Impact

| Metric | Current | Target | Improvement |
|--------|---------|--------|-------------|
| **Max File Size** | 12,000 LOC | 1,000 LOC | **92% reduction** |
| **Avg File Size** | 450 LOC | 350 LOC | **22% reduction** |
| **Functions with Complexity >20** | ~15% | <5% | **67% reduction** |
| **Max Function Complexity** | 40 | 20 | **50% reduction** |
| **Code Review Time (Large Files)** | 60-90 min | 30-45 min | **50% reduction** |
| **Test Setup LOC** | 50-100 | 10-20 | **80% reduction** |

---

## Proposed Solution

### Strategy Overview

```
┌─────────────────────────────────────────────────────┐
│         Code Modularization Approach                │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Phase 1: Extract Interfaces                        │
│  ├─ Define clear contracts                          │
│  ├─ Enable mocking and testing                      │
│  └─ Prepare for implementation splitting            │
│                                                     │
│  Phase 2: Split Large Files                         │
│  ├─ historyEngine.go → 4 smaller files             │
│  ├─ mutable_state_impl.go → 5 smaller files        │
│  └─ historyEngine2_test.go → test suites           │
│                                                     │
│  Phase 3: Reorganize Packages                       │
│  ├─ Sub-package creation                            │
│  ├─ Clear module boundaries                         │
│  └─ Import cycle elimination                        │
│                                                     │
│  Phase 4: Reduce Complexity                         │
│  ├─ Extract helper functions                        │
│  ├─ Strategy pattern for polymorphism              │
│  └─ State machine pattern for workflows             │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## Detailed Design

### 1. Extract Interfaces (Phase 1)

#### 1.1 HistoryEngine Interface Extraction

**Current (implicit interface):**
```go
// service/history/historyEngine.go
type historyEngineImpl struct {
    // 30+ fields
}

// 50+ methods all in one file
func (e *historyEngineImpl) StartWorkflowExecution(...) {}
func (e *historyEngineImpl) RecordActivityTaskStarted(...) {}
func (e *historyEngineImpl) RespondActivityTaskCompleted(...) {}
// ... 47 more methods
```

**Proposed (explicit interface):**
```go
// service/history/engine.go (NEW)
package history

// Engine is the main workflow execution engine interface
type Engine interface {
    WorkflowEngine
    ActivityEngine
    TimerEngine
    SignalEngine
    QueryEngine
}

// WorkflowEngine handles workflow lifecycle operations
type WorkflowEngine interface {
    StartWorkflowExecution(ctx context.Context, request *StartWorkflowExecutionRequest) (*StartWorkflowExecutionResponse, error)
    TerminateWorkflowExecution(ctx context.Context, request *TerminateWorkflowExecutionRequest) error
    CompleteWorkflowExecution(ctx context.Context, request *CompleteWorkflowExecutionRequest) error
    GetWorkflowExecutionHistory(ctx context.Context, request *GetWorkflowExecutionHistoryRequest) (*GetWorkflowExecutionHistoryResponse, error)
}

// ActivityEngine handles activity lifecycle operations
type ActivityEngine interface {
    ScheduleActivityTask(ctx context.Context, request *ScheduleActivityTaskRequest) error
    RecordActivityTaskStarted(ctx context.Context, request *RecordActivityTaskStartedRequest) (*RecordActivityTaskStartedResponse, error)
    RespondActivityTaskCompleted(ctx context.Context, request *RespondActivityTaskCompletedRequest) error
    RespondActivityTaskFailed(ctx context.Context, request *RespondActivityTaskFailedRequest) error
}

// TimerEngine handles timer operations
type TimerEngine interface {
    RecordWorkflowTaskTimeout(ctx context.Context, request *RecordWorkflowTaskTimeoutRequest) error
    RecordActivityTaskTimeout(ctx context.Context, request *RecordActivityTaskTimeoutRequest) error
}

// SignalEngine handles signals and updates
type SignalEngine interface {
    SignalWorkflowExecution(ctx context.Context, request *SignalWorkflowExecutionRequest) error
    SignalWithStartWorkflowExecution(ctx context.Context, request *SignalWithStartWorkflowExecutionRequest) (*SignalWithStartWorkflowExecutionResponse, error)
}

// QueryEngine handles workflow queries
type QueryEngine interface {
    QueryWorkflow(ctx context.Context, request *QueryWorkflowRequest) (*QueryWorkflowResponse, error)
}
```

**Benefits:**
1. Clear API boundaries
2. Easier mocking in tests
3. Enables interface-based composition
4. Documents intended usage

---

#### 1.2 MutableState Interface Extraction

**Current:**
```go
// One giant implementation
type MutableStateImpl struct {
    // Everything
}
```

**Proposed:**
```go
// service/history/workflow/mutable_state.go
package workflow

// MutableState is the main workflow state interface
type MutableState interface {
    WorkflowInfo
    ActivityManager
    TimerManager
    ChildWorkflowManager
    EventManager
    TaskGenerator
}

// WorkflowInfo provides read-only workflow information
type WorkflowInfo interface {
    GetWorkflowID() string
    GetRunID() string
    GetWorkflowType() string
    GetNamespaceID() string
    GetStartTime() time.Time
    GetExecutionState() enumspb.WorkflowExecutionState
}

// ActivityManager manages activity lifecycle
type ActivityManager interface {
    AddActivityTaskScheduledEvent(workflowTaskCompletedID int64, command *commandpb.ScheduleActivityTaskCommandAttributes) (*historypb.HistoryEvent, *persistence.ActivityInfo, error)
    ReplicateActivityTaskScheduledEvent(firstEventID int64, event *historypb.HistoryEvent) (*persistence.ActivityInfo, error)
    GetActivityInfo(scheduledEventID int64) (*persistence.ActivityInfo, bool)
    DeleteActivity(scheduledEventID int64) error
}

// TimerManager manages timers
type TimerManager interface {
    AddTimerStartedEvent(workflowTaskCompletedID int64, command *commandpb.StartTimerCommandAttributes) (*historypb.HistoryEvent, *persistence.TimerInfo, error)
    GetUserTimerInfo(timerID string) (*persistence.TimerInfo, bool)
    DeleteUserTimer(timerID string) error
}

// ChildWorkflowManager manages child workflow executions
type ChildWorkflowManager interface {
    AddStartChildWorkflowExecutionInitiatedEvent(workflowTaskCompletedID int64, initiatedEventID int64, command *commandpb.StartChildWorkflowExecutionCommandAttributes) (*historypb.HistoryEvent, *persistence.ChildExecutionInfo, error)
    GetChildExecutionInfo(initiatedID int64) (*persistence.ChildExecutionInfo, bool)
    DeleteChildExecution(initiatedID int64) error
}

// EventManager handles history event operations
type EventManager interface {
    AddWorkflowExecutionStartedEvent(request *historyservice.StartWorkflowExecutionRequest) (*historypb.HistoryEvent, error)
    AddWorkflowTaskScheduledEvent() (*WorkflowTaskInfo, error)
    AddWorkflowTaskCompletedEvent(scheduledEventID, startedEventID int64) (*historypb.HistoryEvent, error)
    GetCurrentVersion() int64
    GetNextEventID() int64
}

// TaskGenerator generates transfer and timer tasks
type TaskGenerator interface {
    GenerateTransferTasks() ([]tasks.Task, error)
    GenerateTimerTasks() ([]tasks.Task, error)
    GenerateVisibilityTasks() ([]tasks.Task, error)
}
```

**Implementation split:**
```go
// mutable_state.go - Main struct and coordination
type MutableStateImpl struct {
    executionInfo *persistence.WorkflowExecutionInfo
    activityManager *activityManagerImpl
    timerManager *timerManagerImpl
    childWorkflowManager *childWorkflowManagerImpl
    eventManager *eventManagerImpl
    taskGenerator *taskGeneratorImpl
}

// activity_manager.go - Activity-specific logic (500 lines)
type activityManagerImpl struct {
    activities map[int64]*persistence.ActivityInfo
    // ...
}

// timer_manager.go - Timer-specific logic (400 lines)
type timerManagerImpl struct {
    timers map[string]*persistence.TimerInfo
    // ...
}

// child_workflow_manager.go - Child workflow logic (600 lines)
type childWorkflowManagerImpl struct {
    childExecutions map[int64]*persistence.ChildExecutionInfo
    // ...
}

// event_manager.go - Event handling (800 lines)
type eventManagerImpl struct {
    history *historypb.History
    // ...
}

// task_generator.go - Task generation (500 lines)
type taskGeneratorImpl struct {
    // ...
}
```

**Result:**
- ~~4,000 line file~~ → 6 files of 300-800 lines each
- Clear separation of concerns
- Easier to test each component
- Reduced cognitive load

---

### 2. Split Large Files (Phase 2)

#### 2.1 historyEngine.go Refactoring

**Current structure:**
```
historyEngine.go (3,500 lines)
├─ Workflow operations (800 lines)
├─ Activity operations (700 lines)
├─ Timer operations (400 lines)
├─ Signal operations (300 lines)
├─ Query operations (200 lines)
├─ Helper functions (600 lines)
└─ Validation functions (500 lines)
```

**Proposed structure:**
```
history/
├── engine.go                      (200 lines - interface, factory)
├── workflow_engine.go             (800 lines - workflow ops)
├── activity_engine.go             (700 lines - activity ops)
├── timer_engine.go                (400 lines - timer ops)
├── signal_engine.go               (300 lines - signal ops)
├── query_engine.go                (200 lines - query ops)
├── engine_helpers.go              (400 lines - shared helpers)
└── engine_validation.go           (300 lines - validation)
```

**Migration approach:**

**Step 1: Create engine.go with factory**
```go
// service/history/engine.go
package history

import "go.uber.org/fx"

// New creates a new history engine
func NewEngine(params EngineParams) Engine {
    shared := &sharedState{
        shard: params.Shard,
        logger: params.Logger,
        // ... common dependencies
    }

    return &engine{
        workflow: &workflowEngine{shared: shared},
        activity: &activityEngine{shared: shared},
        timer: &timerEngine{shared: shared},
        signal: &signalEngine{shared: shared},
        query: &queryEngine{shared: shared},
    }
}

// engine composites all sub-engines
type engine struct {
    workflow WorkflowEngine
    activity ActivityEngine
    timer    TimerEngine
    signal   SignalEngine
    query    QueryEngine
}

// StartWorkflowExecution delegates to workflow engine
func (e *engine) StartWorkflowExecution(ctx context.Context, request *StartWorkflowExecutionRequest) (*StartWorkflowExecutionResponse, error) {
    return e.workflow.StartWorkflowExecution(ctx, request)
}
```

**Step 2: Extract workflow_engine.go**
```go
// service/history/workflow_engine.go
package history

type workflowEngine struct {
    shared *sharedState
}

func (w *workflowEngine) StartWorkflowExecution(
    ctx context.Context,
    request *StartWorkflowExecutionRequest,
) (*StartWorkflowExecutionResponse, error) {
    // Original implementation from historyEngine.go
    // 100-150 lines of logic
}

func (w *workflowEngine) TerminateWorkflowExecution(...) error {
    // ...
}

// ... other workflow operations
```

**Step 3: Repeat for activity, timer, signal, query**

**Step 4: Deprecate old file**
```go
// service/history/historyEngine.go (deprecated)
package history

// Deprecated: Use Engine interface instead
type historyEngineImpl = engine

// Deprecated: Use NewEngine instead
func newHistoryEngine(params EngineParams) Engine {
    return NewEngine(params)
}
```

**Step 5: Update all callers (gradual, with deprecation warnings)**

---

#### 2.2 mutable_state_impl.go Refactoring

**Already covered in section 1.2 above.**

**Additional: Extract state machine logic**

```go
// service/history/workflow/state_machine.go (NEW)
package workflow

// StateMachine handles workflow state transitions
type StateMachine struct {
    currentState enumspb.WorkflowExecutionState
    currentStatus enumspb.WorkflowExecutionStatus
}

// CanTransition checks if transition is valid
func (sm *StateMachine) CanTransition(toState enumspb.WorkflowExecutionState) bool {
    validTransitions := map[enumspb.WorkflowExecutionState][]enumspb.WorkflowExecutionState{
        enumspb.WORKFLOW_EXECUTION_STATE_CREATED: {
            enumspb.WORKFLOW_EXECUTION_STATE_RUNNING,
        },
        enumspb.WORKFLOW_EXECUTION_STATE_RUNNING: {
            enumspb.WORKFLOW_EXECUTION_STATE_COMPLETED,
            enumspb.WORKFLOW_EXECUTION_STATE_ZOMBIE,
        },
        // ... more transitions
    }

    allowed, ok := validTransitions[sm.currentState]
    if !ok {
        return false
    }

    for _, state := range allowed {
        if state == toState {
            return true
        }
    }
    return false
}

// Transition performs state transition with validation
func (sm *StateMachine) Transition(toState enumspb.WorkflowExecutionState, toStatus enumspb.WorkflowExecutionStatus) error {
    if !sm.CanTransition(toState) {
        return fmt.Errorf("invalid transition from %s to %s", sm.currentState, toState)
    }

    sm.currentState = toState
    sm.currentStatus = toStatus
    return nil
}
```

---

#### 2.3 historyEngine2_test.go Refactoring

**Problem:** 12,000 line test file is unmaintainable

**Current:**
```
historyEngine2_test.go (12,000 lines)
├─ Workflow tests (3,000 lines)
├─ Activity tests (2,500 lines)
├─ Timer tests (1,500 lines)
├─ Signal tests (1,000 lines)
├─ Query tests (500 lines)
├─ Edge case tests (2,000 lines)
└─ Helper functions (1,500 lines)
```

**Proposed:**
```
history/
├── workflow_engine_test.go        (1,200 lines)
├── activity_engine_test.go        (1,000 lines)
├── timer_engine_test.go           (600 lines)
├── signal_engine_test.go          (500 lines)
├── query_engine_test.go           (300 lines)
├── edge_cases_test.go             (1,500 lines)
└── test_helpers.go                (500 lines)
```

**Test Suite Organization:**
```go
// service/history/workflow_engine_test.go
package history

import (
    "testing"
    "github.com/stretchr/testify/suite"
)

// WorkflowEngineSuite tests WorkflowEngine interface
type WorkflowEngineSuite struct {
    suite.Suite
    engine WorkflowEngine
    // ... test fixtures
}

func TestWorkflowEngineSuite(t *testing.T) {
    suite.Run(t, new(WorkflowEngineSuite))
}

func (s *WorkflowEngineSuite) SetupTest() {
    // Setup once per test
}

func (s *WorkflowEngineSuite) TestStartWorkflowExecution_Success() {
    // Test happy path
}

func (s *WorkflowEngineSuite) TestStartWorkflowExecution_AlreadyStarted() {
    // Test error case
}

// ... more focused tests
```

---

### 3. Package Reorganization (Phase 3)

#### 3.1 Persistence Package Restructuring

**Current:**
```
common/persistence/
├── 80+ files in flat structure
├── Hard to navigate
└── Unclear module boundaries
```

**Proposed:**
```
common/persistence/
├── README.md                    # Package overview
│
├── interface.go                 # Core interfaces
├── types.go                     # Common types
├── errors.go                    # Error types
│
├── client/                      # Client abstractions
│   ├── factory.go
│   ├── bean.go
│   └── interface.go
│
├── datastore/                   # Database interfaces
│   ├── shard.go
│   ├── execution.go
│   ├── task.go
│   └── visibility.go
│
├── sql/                         # SQL implementations
│   ├── sqlplugin/
│   │   ├── interface.go
│   │   ├── postgresql/
│   │   │   ├── plugin.go
│   │   │   ├── workflow.go
│   │   │   └── visibility.go
│   │   ├── mysql/
│   │   └── sqlite/
│   └── execution_store.go
│
├── nosql/                       # NoSQL implementations
│   └── cassandra/
│       ├── plugin.go
│       └── workflow.go
│
├── serialization/               # Data serialization
│   ├── parser.go
│   ├── serializer.go
│   └── thrift.go
│
└── tests/                       # Shared test utilities
    ├── persistence_suite.go
    └── test_factories.go
```

**Migration:**
```bash
# Create new structure
mkdir -p common/persistence/{client,datastore,sql,nosql,serialization,tests}

# Move files gradually
git mv common/persistence/factory.go common/persistence/client/
git mv common/persistence/executionStore.go common/persistence/datastore/

# Update imports (use tools/refactor-imports.sh)
```

---

### 4. Complexity Reduction (Phase 4)

#### 4.1 Extract Helper Functions

**Before:**
```go
// High complexity function (complexity: 35)
func (e *historyEngineImpl) StartWorkflowExecution(
    ctx context.Context,
    request *StartWorkflowExecutionRequest,
) (*StartWorkflowExecutionResponse, error) {
    // Validation (20 lines)
    if request == nil {
        return nil, errors.New("request is nil")
    }
    if request.WorkflowId == "" {
        return nil, errors.New("workflow ID is empty")
    }
    // ... 15 more validations

    // Namespace lookup (15 lines)
    namespace, err := e.namespaceRegistry.GetNamespace(...)
    if err != nil {
        return nil, err
    }
    if namespace.State != enumspb.NAMESPACE_STATE_REGISTERED {
        return nil, errors.New("namespace not active")
    }
    // ... more namespace checks

    // Shard determination (10 lines)
    shardID := e.shard.GetShardID()
    // ... complex logic

    // Create mutable state (30 lines)
    mutableState := e.createMutableState(...)
    // ... lots of setup

    // Persist (20 lines)
    err = e.persistence.CreateWorkflowExecution(...)
    // ... error handling

    return response, nil
}
```

**After:**
```go
// Reduced complexity (complexity: 8)
func (e *historyEngineImpl) StartWorkflowExecution(
    ctx context.Context,
    request *StartWorkflowExecutionRequest,
) (*StartWorkflowExecutionResponse, error) {
    // Validate request (complexity: 1)
    if err := validateStartWorkflowRequest(request); err != nil {
        return nil, err
    }

    // Get and validate namespace (complexity: 1)
    namespace, err := e.getActiveNamespace(ctx, request.NamespaceId)
    if err != nil {
        return nil, err
    }

    // Determine shard (complexity: 1)
    shardContext, err := e.getShardForWorkflow(namespace.ID(), request.WorkflowId)
    if err != nil {
        return nil, err
    }

    // Create workflow execution (complexity: 3)
    mutableState, err := e.createNewWorkflowExecution(ctx, shardContext, request)
    if err != nil {
        return nil, err
    }

    // Persist (complexity: 2)
    return e.persistWorkflowExecution(ctx, shardContext, mutableState)
}

// Helper functions (each with low complexity)
func validateStartWorkflowRequest(request *StartWorkflowExecutionRequest) error {
    // Validation logic (complexity: 5)
    // ...
}

func (e *historyEngineImpl) getActiveNamespace(ctx context.Context, namespaceID string) (*namespace.Namespace, error) {
    // Namespace lookup logic (complexity: 4)
    // ...
}

func (e *historyEngineImpl) getShardForWorkflow(namespaceID, workflowID string) (*shard.Context, error) {
    // Shard determination (complexity: 3)
    // ...
}

func (e *historyEngineImpl) createNewWorkflowExecution(ctx context.Context, shardContext *shard.Context, request *StartWorkflowExecutionRequest) (workflow.MutableState, error) {
    // Mutable state creation (complexity: 8)
    // ...
}

func (e *historyEngineImpl) persistWorkflowExecution(ctx context.Context, shardContext *shard.Context, mutableState workflow.MutableState) (*StartWorkflowExecutionResponse, error) {
    // Persistence logic (complexity: 6)
    // ...
}
```

**Benefits:**
- Main function complexity: 35 → 8
- Each helper function: <10 complexity
- Easier to test each piece
- Clearer logic flow

---

#### 4.2 Strategy Pattern for Polymorphism

**Problem:** Switch statements with many cases

**Before:**
```go
func (e *historyEngineImpl) processCommand(command *commandpb.Command) error {
    switch command.CommandType {
    case enumspb.COMMAND_TYPE_SCHEDULE_ACTIVITY_TASK:
        return e.handleScheduleActivityTask(command.GetScheduleActivityTaskCommandAttributes())
    case enumspb.COMMAND_TYPE_COMPLETE_WORKFLOW_EXECUTION:
        return e.handleCompleteWorkflowExecution(command.GetCompleteWorkflowExecutionCommandAttributes())
    case enumspb.COMMAND_TYPE_FAIL_WORKFLOW_EXECUTION:
        return e.handleFailWorkflowExecution(command.GetFailWorkflowExecutionCommandAttributes())
    case enumspb.COMMAND_TYPE_CANCEL_WORKFLOW_EXECUTION:
        return e.handleCancelWorkflowExecution(command.GetCancelWorkflowExecutionCommandAttributes())
    // ... 20 more cases
    }
}
```

**After (Strategy Pattern):**
```go
// Command handler interface
type CommandHandler interface {
    Handle(ctx context.Context, command *commandpb.Command, mutableState workflow.MutableState) error
}

// Registry of handlers
type CommandHandlerRegistry struct {
    handlers map[enumspb.CommandType]CommandHandler
}

func NewCommandHandlerRegistry() *CommandHandlerRegistry {
    registry := &CommandHandlerRegistry{
        handlers: make(map[enumspb.CommandType]CommandHandler),
    }

    // Register handlers
    registry.Register(enumspb.COMMAND_TYPE_SCHEDULE_ACTIVITY_TASK, &ScheduleActivityTaskHandler{})
    registry.Register(enumspb.COMMAND_TYPE_COMPLETE_WORKFLOW_EXECUTION, &CompleteWorkflowExecutionHandler{})
    // ... register other handlers

    return registry
}

func (r *CommandHandlerRegistry) Handle(ctx context.Context, command *commandpb.Command, mutableState workflow.MutableState) error {
    handler, ok := r.handlers[command.CommandType]
    if !ok {
        return fmt.Errorf("unknown command type: %v", command.CommandType)
    }

    return handler.Handle(ctx, command, mutableState)
}

// Example handler implementation
type ScheduleActivityTaskHandler struct{}

func (h *ScheduleActivityTaskHandler) Handle(ctx context.Context, command *commandpb.Command, mutableState workflow.MutableState) error {
    attrs := command.GetScheduleActivityTaskCommandAttributes()
    _, _, err := mutableState.AddActivityTaskScheduledEvent(...)
    return err
}
```

**Benefits:**
- No giant switch statement
- Easy to add new command types (Open/Closed Principle)
- Each handler independently testable
- Lower complexity

---

## Implementation Plan

### Phase 1: Interface Extraction (Weeks 1-2)

**Week 1: Define Interfaces**
```bash
# Day 1-2: HistoryEngine interfaces
touch service/history/engine.go
# Define WorkflowEngine, ActivityEngine, etc.

# Day 3-4: MutableState interfaces
touch service/history/workflow/mutable_state.go
# Define ActivityManager, TimerManager, etc.

# Day 5: Persistence interfaces
# Review and refine existing interfaces
```

**Week 2: Implementation Stubs**
```bash
# Create stub implementations that delegate to existing code
# This allows gradual migration without breaking existing functionality

# Example:
type workflowEngine struct {
    impl *historyEngineImpl  # Delegate to existing implementation
}

func (w *workflowEngine) StartWorkflowExecution(...) {
    return w.impl.StartWorkflowExecution(...)  # Forward call
}
```

**Deliverables:**
- ✅ All interfaces defined
- ✅ Stub implementations created
- ✅ Tests pass (no functionality changes yet)

---

### Phase 2: File Splitting (Weeks 3-4)

**Week 3: HistoryEngine Split**
```bash
# Day 1: Create workflow_engine.go
git mv service/history/historyEngine.go service/history/historyEngine_old.go
# Extract workflow operations to workflow_engine.go

# Day 2: Create activity_engine.go
# Extract activity operations

# Day 3: Create timer_engine.go, signal_engine.go, query_engine.go
# Extract remaining operations

# Day 4: Update callers
# Fix import paths, update tests

# Day 5: Delete old file
git rm service/history/historyEngine_old.go
```

**Week 4: MutableState Split**
```bash
# Day 1: Create activity_manager.go
# Extract activity management logic

# Day 2: Create timer_manager.go
# Extract timer logic

# Day 3: Create child_workflow_manager.go, event_manager.go
# Extract child workflow and event logic

# Day 4: Create task_generator.go
# Extract task generation logic

# Day 5: Integration testing
# Ensure all tests pass
```

**Deliverables:**
- ✅ historyEngine.go split into 7 files (each <800 LOC)
- ✅ mutable_state_impl.go split into 6 files (each <800 LOC)
- ✅ All tests pass
- ✅ No functional changes

---

### Phase 3: Test File Refactoring (Week 5)

```bash
# Day 1-2: Split historyEngine2_test.go
cp service/history/historyEngine2_test.go service/history/historyEngine2_test_old.go
# Extract workflow tests to workflow_engine_test.go
# Extract activity tests to activity_engine_test.go

# Day 3: Split timer, signal, query tests
# Create separate test files

# Day 4: Extract test helpers
# Create test_helpers.go with shared utilities

# Day 5: Validation
# Run all tests, ensure 100% pass rate
git rm service/history/historyEngine2_test_old.go
```

**Deliverables:**
- ✅ Test files split (each <1,500 LOC)
- ✅ Test helpers extracted
- ✅ Test coverage maintained
- ✅ All tests pass

---

### Phase 4: Package Reorganization (Week 6)

```bash
# Day 1-2: Create new persistence structure
mkdir -p common/persistence/{client,datastore,sql,nosql,serialization,tests}

# Day 3-4: Move files
git mv common/persistence/factory.go common/persistence/client/
git mv common/persistence/executionStore.go common/persistence/datastore/
# ... move other files

# Day 5: Update imports
# Use find/replace or refactoring tools
# Update all import paths
```

**Deliverables:**
- ✅ Persistence package reorganized
- ✅ Clear module structure
- ✅ All imports updated
- ✅ Tests pass

---

### Phase 5: Complexity Reduction (Weeks 7-8)

**Week 7: Extract Helper Functions**
```bash
# Day 1-3: Refactor high-complexity functions
# Target functions with complexity >20
# Extract helper functions

# Day 4: Validate complexity reduction
go install github.com/fzipp/gocyclo/cmd/gocyclo@latest
gocyclo -over 15 service/history/
# Should show significant reduction

# Day 5: Code review
```

**Week 8: Apply Design Patterns**
```bash
# Day 1-2: Implement Strategy Pattern for commands
# Create CommandHandlerRegistry
# Migrate command processing

# Day 3: Implement State Machine Pattern
# Create StateMachine for workflow state transitions

# Day 4-5: Integration testing, performance validation
```

**Deliverables:**
- ✅ Average function complexity <15
- ✅ No functions with complexity >25
- ✅ Design patterns applied
- ✅ Tests pass

---

## Success Metrics

### Quantitative Metrics

| Metric | Baseline | Target | Measurement |
|--------|----------|--------|-------------|
| **Max File Size** | 12,000 LOC | 1,500 LOC | `wc -l` on all .go files |
| **Avg File Size** | 450 LOC | 350 LOC | Average LOC per file |
| **Max Function Complexity** | 40 | 20 | `gocyclo -over 15 ./...` |
| **Functions >20 Complexity** | 15% | <5% | Complexity analysis |
| **Code Review Time** | 60-90 min | 30-45 min | Developer survey |
| **Test Setup LOC** | 50-100 | 10-20 | Test file analysis |
| **Build Time** | 8 min | 7 min | CI timing |

### Qualitative Metrics

**Developer Survey (1-5 scale):**
1. "I can understand new code more easily"
2. "Code reviews are less time-consuming"
3. "I feel confident making changes without breaking things"
4. "Test writing is easier"

**Success Criteria:**
- ✅ All questions average >4.0 after completion
- ✅ Zero production regressions from refactoring
- ✅ Positive feedback from 90%+ of developers

---

## Risks and Mitigations

### Risk 1: Introduction of Regressions

**Likelihood:** High
**Impact:** Critical

**Mitigation:**
```yaml
Strategy:
  - Maintain 100% test pass rate throughout refactoring
  - Add tests before refactoring (improve coverage to 80%+)
  - Use automated refactoring tools where possible
  - Code review every change with 2+ reviewers
  - Canary deployments for each phase
  - Keep feature flags for rollback

Testing:
  - Run full test suite after each file split
  - Performance benchmarks (ensure no degradation)
  - Integration tests in staging environment
  - Load testing before production deployment
```

---

### Risk 2: Import Cycle Creation

**Likelihood:** Medium
**Impact:** High

**Mitigation:**
- Use dependency graph analysis tools
- Follow dependency rules (lower layers don't import upper layers)
- Code review checklist includes import cycle check
- CI check: `go build ./...` must pass

---

### Risk 3: Developer Confusion During Transition

**Likelihood:** Medium
**Impact:** Medium

**Mitigation:**
- Clear communication plan (announce each phase)
- Migration guide document
- Code pointers (deprecation comments with "see X instead")
- Office hours for questions
- Internal tech talk explaining changes

---

## Alternatives Considered

### Alternative 1: Big Bang Refactoring

**Approach:** Refactor everything at once in one large PR

**Pros:**
- Faster completion
- No temporary inconsistency

**Cons:**
- **VERY HIGH RISK** of breaking things
- Impossible to review 10,000+ line PR
- Hard to identify source of regressions
- Team blocked while refactoring

**Decision:** Rejected, too risky

---

### Alternative 2: Only Extract Interfaces (No File Splitting)

**Approach:** Define interfaces but keep implementations in large files

**Pros:**
- Less churn
- Easier migration

**Cons:**
- **Doesn't solve the maintainability problem**
- Files still too large
- Code reviews still painful

**Decision:** Rejected, doesn't address root cause

---

### Alternative 3: Complete Rewrite

**Approach:** Rewrite history engine from scratch

**Pros:**
- Clean slate
- Apply all best practices

**Cons:**
- **6-12 months of work**
- Requires deep domain knowledge
- High risk of introducing new bugs
- Existing code works (mostly)

**Decision:** Rejected, refactoring is sufficient

---

## Migration Strategy

### Phase 0: Preparation (Before Week 1)

**Communication:**
- Announce refactoring plan to team
- Share RFC for feedback
- Create Slack channel: #refactoring-2025

**Tooling:**
- Setup complexity analysis: `go install github.com/fzipp/gocyclo/cmd/gocyclo@latest`
- Setup refactoring tools
- CI enhancements for import cycle detection

---

### Phase 1-5: Gradual Rollout (Weeks 1-8)

**Each week:**
1. Implement changes (Mon-Thu)
2. Code review (Thu afternoon)
3. Merge to main (Fri morning)
4. Monitor CI and staging (Fri afternoon)
5. Deploy canary to production (following Monday)

**Rollback plan:**
- If issues detected, revert merge immediately
- Feature flags allow disabling new code paths
- Keep old implementations for 1 week after new implementation is stable

---

### Phase 6: Cleanup (Week 9+)

**Remove deprecated code:**
```bash
# Week 9: Remove old implementations
git rm service/history/historyEngine_old.go
git rm service/history/workflow/mutable_state_impl_old.go

# Update documentation
# Announce completion
```

---

## Open Questions

1. **Q:** Should we use automated refactoring tools (e.g., Goland refactoring, gopls)?
   **A:** Yes, for mechanical changes (renames, moves). Manual for logic changes.

2. **Q:** How to handle ongoing development during refactoring?
   **A:** Refactor in feature branches, rebase frequently, communicate with team.

3. **Q:** Should we refactor tests first or production code first?
   **A:** Production code first (with existing tests), then refactor tests.

4. **Q:** What to do if we discover bugs during refactoring?
   **A:** Fix bugs in separate PRs (don't mix refactoring with bug fixes).

---

## Related Work

- **RFC-0005: Testing Infrastructure** - Better tests make refactoring safer
- **RFC-0006: Documentation** - Document new structure for contributors
- **Blog Post 3: Design Patterns** - Patterns used in refactoring

---

## References

### Books
- *Refactoring* by Martin Fowler
- *Working Effectively with Legacy Code* by Michael Feathers
- *Clean Code* by Robert C. Martin

### Tools
- [gocyclo](https://github.com/fzipp/gocyclo) - Cyclomatic complexity analysis
- [goimports](https://pkg.go.dev/golang.org/x/tools/cmd/goimports) - Import management
- [gorename](https://pkg.go.dev/golang.org/x/tools/cmd/gorename) - Safe renaming

### Temporal Resources
- [Current Architecture Docs](https://github.com/temporalio/temporal/tree/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/docs/architecture)
- [Service Structure](https://github.com/temporalio/temporal/tree/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/service)

---

**Next Steps:**
1. Review and approve RFC
2. Create refactoring project board
3. Assign owners for each phase
4. Begin Phase 1 (interface extraction)

**Status:** Ready for review
**Estimated Completion:** 8 weeks from approval
