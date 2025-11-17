# RFC-0005: Testing Infrastructure Enhancements

**Status:** Draft
**Effort:** 2-3 weeks
**Impact:** High
**Author:** Analysis Team
**Created:** 2025-11-16
**Commit SHA:** `0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3`

---

## Summary

Enhance Temporal's testing infrastructure to achieve >80% test coverage, reduce CI execution time from 90+ minutes to <45 minutes, and eliminate flaky tests (reduce from 10% to <1% failure rate). This includes creating better test utilities, optimizing CI parallelization, and systematically addressing coverage gaps.

---

## Motivation

### Current Problems

**Problem 1: Test Coverage Gaps (~70%)**
```bash
# Current coverage estimation
Unit Tests:     ~70% coverage (estimated)
Integration:    ~60% coverage (persistence, tools)
Functional:     High coverage for happy paths, gaps in edge cases
```

**Missing Coverage Areas:**
- Error handling paths (especially persistence layer failures)
- Edge cases in workflow state machine transitions
- Concurrent workflow execution scenarios
- Advanced features (Nexus, worker versioning)
- Disaster recovery paths
- Cross-datacenter replication edge cases

**Problem 2: Slow CI Execution (90+ minutes)**
```
Current CI times:
├─ Unit Tests:         10-15 min (3 shards, could be faster)
├─ Integration Tests:  20-30 min per database (4 databases = 80-120 min)
├─ Functional Tests:   60-90 min per database
└─ Total:              90-150 min (worst case)
```

**Impact:**
- Slow feedback loop (developers wait 1.5-2.5 hours for CI)
- Higher context switching cost
- Delayed merge times
- Discouraged from running full test suite locally

**Problem 3: Flaky Tests (~10% failure rate)**
```bash
# Common flaky test patterns
- Race conditions in concurrent tests
- Timeout sensitivity (hardcoded 5s, 10s timeouts)
- Database cleanup issues (leftover state)
- Time-dependent assertions (time.Now() comparisons)
- Goroutine leaks causing subsequent test failures
```

**Example Flaky Test Pattern:**
```go
// From: service/history/historyEngine2_test.go (line ~8500)
// Problem: Hardcoded timeout, race condition
func (s *engine2Suite) TestActivityHeartbeatTimeout() {
    // ... setup ...
    time.Sleep(5 * time.Second)  // ❌ Flaky: depends on system load
    s.Eventually(func() bool {
        // Check activity timed out
    }, 10*time.Second, 100*time.Millisecond)  // ❌ Sometimes fails
}
```

**Problem 4: Limited Test Utilities**

Current test utilities are good but could be improved:
- `testvars` - Deterministic test variables (good)
- `taskpoller` - Task polling simulation (good, but limited)
- Clock mocking - Limited, not consistently used
- Database setup - Each test suite reimplements setup logic

**Missing:**
- Workflow execution builders (reduce boilerplate)
- Advanced task queue simulation
- Chaos testing utilities
- Performance regression detection

**Problem 5: Local Testing Experience**

Running full test suite locally is painful:
```bash
# Current experience
make test-all  # Takes 60-90 minutes, requires 4 databases
make test      # Only unit tests, misses integration issues
```

Developers often skip integration/functional tests locally, discovering issues only in CI.

---

## Proposed Solution

### Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│              Enhanced Testing Framework                 │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌───────────────┐  ┌────────────────┐  ┌───────────┐ │
│  │ Test Utilities│  │ CI Optimization│  │  Coverage │ │
│  ├───────────────┤  ├────────────────┤  ├───────────┤ │
│  │• WorkflowBld  │  │• Smart Sharding│  │• Gap Anal │ │
│  │• Clock Mock   │  │• Caching       │  │• 80% Goal │ │
│  │• Chaos Inject │  │• Paralleliztn  │  │• Reports  │ │
│  └───────────────┘  └────────────────┘  └───────────┘ │
│                                                         │
│  ┌───────────────────────────────────────────────────┐ │
│  │         Flaky Test Elimination                    │ │
│  ├───────────────────────────────────────────────────┤ │
│  │• Deterministic clocks                             │ │
│  │• Proper cleanup hooks                             │ │
│  │• Race detector always on                          │ │
│  │• Goroutine leak detection                         │ │
│  └───────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

---

## Detailed Design

### 1. Enhanced Test Utilities

#### 1.1 Workflow Execution Builder

**Problem:** Tests have 50-100 lines of boilerplate to set up workflows.

**Solution:** Fluent builder API

**Implementation:**
```go
// File: common/testing/workflowbuilder/builder.go
package workflowbuilder

import (
    "time"
    enumspb "go.temporal.io/api/enums/v1"
    historypb "go.temporal.io/api/history/v1"
)

// WorkflowBuilder provides fluent API for test workflow setup
type WorkflowBuilder struct {
    events      []*historypb.HistoryEvent
    nextEventID int64
    workflowID  string
    runID       string
    taskQueue   string
}

func New() *WorkflowBuilder {
    return &WorkflowBuilder{
        events:      make([]*historypb.HistoryEvent, 0),
        nextEventID: 1,
    }
}

// Start adds WorkflowExecutionStarted event
func (b *WorkflowBuilder) Start(workflowType string) *WorkflowBuilder {
    b.events = append(b.events, &historypb.HistoryEvent{
        EventId:   b.nextEventID,
        EventType: enumspb.EVENT_TYPE_WORKFLOW_EXECUTION_STARTED,
        Attributes: &historypb.HistoryEvent_WorkflowExecutionStartedEventAttributes{
            WorkflowExecutionStartedEventAttributes: &historypb.WorkflowExecutionStartedEventAttributes{
                WorkflowType: &commonpb.WorkflowType{Name: workflowType},
                TaskQueue:    &taskqueuepb.TaskQueue{Name: b.taskQueue},
            },
        },
    })
    b.nextEventID++
    return b
}

// ScheduleActivity adds ActivityTaskScheduled event
func (b *WorkflowBuilder) ScheduleActivity(activityID, activityType string) *WorkflowBuilder {
    b.events = append(b.events, &historypb.HistoryEvent{
        EventId:   b.nextEventID,
        EventType: enumspb.EVENT_TYPE_ACTIVITY_TASK_SCHEDULED,
        Attributes: &historypb.HistoryEvent_ActivityTaskScheduledEventAttributes{
            ActivityTaskScheduledEventAttributes: &historypb.ActivityTaskScheduledEventAttributes{
                ActivityId:   activityID,
                ActivityType: &commonpb.ActivityType{Name: activityType},
                TaskQueue:    &taskqueuepb.TaskQueue{Name: b.taskQueue},
            },
        },
    })
    b.nextEventID++
    return b
}

// CompleteActivity adds ActivityTaskCompleted event
func (b *WorkflowBuilder) CompleteActivity(scheduledEventID int64, result string) *WorkflowBuilder {
    b.events = append(b.events, &historypb.HistoryEvent{
        EventId:   b.nextEventID,
        EventType: enumspb.EVENT_TYPE_ACTIVITY_TASK_COMPLETED,
        Attributes: &historypb.HistoryEvent_ActivityTaskCompletedEventAttributes{
            ActivityTaskCompletedEventAttributes: &historypb.ActivityTaskCompletedEventAttributes{
                ScheduledEventId: scheduledEventID,
                Result:           payloads.EncodeString(result),
            },
        },
    })
    b.nextEventID++
    return b
}

// Complete adds WorkflowExecutionCompleted event
func (b *WorkflowBuilder) Complete(result string) *WorkflowBuilder {
    b.events = append(b.events, &historypb.HistoryEvent{
        EventId:   b.nextEventID,
        EventType: enumspb.EVENT_TYPE_WORKFLOW_EXECUTION_COMPLETED,
        Attributes: &historypb.HistoryEvent_WorkflowExecutionCompletedEventAttributes{
            WorkflowExecutionCompletedEventAttributes: &historypb.WorkflowExecutionCompletedEventAttributes{
                Result: payloads.EncodeString(result),
            },
        },
    })
    b.nextEventID++
    return b
}

// Build returns the history and mutable state
func (b *WorkflowBuilder) Build() (*historypb.History, error) {
    return &historypb.History{
        Events: b.events,
    }, nil
}
```

**Usage Example:**
```go
// Before (50 lines of boilerplate)
func TestWorkflowExecution_Old(t *testing.T) {
    event1 := &historypb.HistoryEvent{...}  // 10 lines
    event2 := &historypb.HistoryEvent{...}  // 10 lines
    event3 := &historypb.HistoryEvent{...}  // 10 lines
    // ... setup continues
}

// After (5 lines)
func TestWorkflowExecution_New(t *testing.T) {
    history, _ := workflowbuilder.New().
        Start("MyWorkflow").
        ScheduleActivity("activity1", "SendEmail").
        CompleteActivity(2, "success").
        Complete("done").
        Build()

    // Test against history
}
```

---

#### 1.2 Deterministic Clock for Testing

**Problem:** Time-based tests are flaky and slow.

**Current Issue:**
```go
// Flaky test pattern
time.Sleep(5 * time.Second)  // ❌ Slow and unreliable
```

**Solution:** Mockable time interface

**Implementation:**
```go
// File: common/clock/clock.go (enhance existing)
package clock

import "time"

// TimeSource is mockable time interface
type TimeSource interface {
    Now() time.Time
    Since(t time.Time) time.Duration
    Until(t time.Time) time.Duration
    Sleep(d time.Duration)
    After(d time.Duration) <-chan time.Time
    NewTimer(d time.Duration) *Timer
    NewTicker(d time.Duration) *Ticker
}

// MockedTimeSource for testing
type MockedTimeSource struct {
    current time.Time
    timers  []*Timer
    mu      sync.Mutex
}

func NewMockedTimeSource() *MockedTimeSource {
    return &MockedTimeSource{
        current: time.Date(2025, 1, 1, 0, 0, 0, 0, time.UTC),
        timers:  make([]*Timer, 0),
    }
}

func (m *MockedTimeSource) Now() time.Time {
    m.mu.Lock()
    defer m.mu.Unlock()
    return m.current
}

// Advance moves time forward deterministically
func (m *MockedTimeSource) Advance(d time.Duration) {
    m.mu.Lock()
    m.current = m.current.Add(d)
    m.mu.Unlock()

    // Fire all timers that should trigger
    m.fireTimers()
}

func (m *MockedTimeSource) fireTimers() {
    m.mu.Lock()
    defer m.mu.Unlock()

    for _, timer := range m.timers {
        if !timer.deadline.After(m.current) {
            select {
            case timer.C <- m.current:
            default:
            }
        }
    }
}
```

**Usage in Tests:**
```go
func TestActivityTimeout_Deterministic(t *testing.T) {
    clock := clock.NewMockedTimeSource()

    // Setup workflow with clock
    engine := newTestEngine(t, clock)

    // Start workflow with 10-second activity timeout
    engine.StartWorkflow(...)

    // Advance clock instead of sleeping
    clock.Advance(11 * time.Second)  // ✅ Instant, deterministic

    // Verify timeout occurred
    require.Equal(t, enumspb.EVENT_TYPE_ACTIVITY_TASK_TIMED_OUT, lastEvent.EventType)
}
```

---

#### 1.3 Chaos Testing Utilities

**Goal:** Test failure scenarios systematically

**Implementation:**
```go
// File: common/testing/chaos/injector.go
package chaos

import (
    "context"
    "errors"
    "math/rand"
)

// FaultInjector injects failures for testing
type FaultInjector struct {
    failureRate float64  // 0.0 - 1.0
    failures    map[string]error
    enabled     bool
}

func NewFaultInjector() *FaultInjector {
    return &FaultInjector{
        failures: make(map[string]error),
        enabled:  false,
    }
}

// InjectPersistenceErrors makes persistence randomly fail
func (f *FaultInjector) InjectPersistenceErrors(rate float64) {
    f.failureRate = rate
    f.failures["persistence"] = errors.New("injected: database unavailable")
}

// ShouldFail returns whether operation should fail
func (f *FaultInjector) ShouldFail(operation string) (bool, error) {
    if !f.enabled {
        return false, nil
    }

    if err, ok := f.failures[operation]; ok {
        if rand.Float64() < f.failureRate {
            return true, err
        }
    }
    return false, nil
}
```

**Usage:**
```go
func TestWorkflowRecovery_PersistenceFailures(t *testing.T) {
    chaos := chaos.NewFaultInjector()
    chaos.InjectPersistenceErrors(0.3)  // 30% failure rate

    engine := newTestEngineWithChaos(t, chaos)

    // Run workflow - should succeed despite failures
    result := engine.StartWorkflow(...)

    require.Equal(t, "completed", result.Status)
    // Verify retries occurred
    require.Greater(t, metrics.Get("persistence_retries"), 0)
}
```

---

### 2. CI Optimization

#### 2.1 Smart Test Sharding

**Current:** Fixed 3 shards for unit tests, sequential database tests

**Proposed:** Dynamic sharding based on test execution time

**Implementation:**
```yaml
# .github/workflows/test-optimized.yml
name: Optimized Tests

on: [push, pull_request]

jobs:
  unit-tests:
    strategy:
      matrix:
        shard: [1, 2, 3, 4, 5]  # Increase from 3 to 5 shards
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run unit tests (smart sharding)
        run: |
          # Use test2json to split tests by package timing
          go test -json -short ./... | \
            go run tools/shard-tests/main.go \
              --shard=${{ matrix.shard }} \
              --total=5 \
              --timing-file=.test-timings.json

      - name: Upload timing data
        uses: actions/upload-artifact@v3
        with:
          name: test-timings-${{ matrix.shard }}
          path: .test-timings.json

  integration-tests:
    strategy:
      matrix:
        db: [cassandra, postgresql, mysql, sqlite]
        shard: [1, 2]  # Shard integration tests per DB
    runs-on: ubuntu-latest
    steps:
      - name: Run integration tests
        run: |
          make test-integration-${{ matrix.db }} \
            SHARD=${{ matrix.shard }} \
            TOTAL_SHARDS=2
```

**Test Sharding Tool:**
```go
// File: tools/shard-tests/main.go
package main

import (
    "encoding/json"
    "os"
    "sort"
)

// ShardTests distributes tests across shards by execution time
func main() {
    shard := flag.Int("shard", 1, "Current shard")
    total := flag.Int("total", 3, "Total shards")
    timingFile := flag.String("timing-file", "", "Test timing data")

    // Load historical test timings
    timings := loadTimings(*timingFile)

    // Sort packages by execution time (longest first)
    sort.Slice(timings, func(i, j int) bool {
        return timings[i].Duration > timings[j].Duration
    })

    // Distribute packages using greedy algorithm (balance shard times)
    shards := distributeTests(timings, *total)

    // Run tests for this shard
    for _, pkg := range shards[*shard-1] {
        runTest(pkg)
    }
}
```

**Expected Result:**
```
Current:  Unit tests 15 min (3 shards, unbalanced)
Optimized: Unit tests 8 min (5 shards, balanced)

Current:  Integration 4×25 min = 100 min (sequential per DB)
Optimized: Integration 4×12 min = 48 min (2 shards per DB)
```

---

#### 2.2 Aggressive Caching

**Cache Layers:**
1. Go build cache
2. Go module cache
3. Docker layer cache
4. Test compilation cache

**Implementation:**
```yaml
# .github/workflows/test-optimized.yml
jobs:
  test:
    steps:
      - name: Cache Go modules
        uses: actions/cache@v3
        with:
          path: |
            ~/.cache/go-build
            ~/go/pkg/mod
          key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}
          restore-keys: |
            ${{ runner.os }}-go-

      - name: Cache test binaries
        uses: actions/cache@v3
        with:
          path: .test-cache
          key: test-binaries-${{ github.sha }}
          restore-keys: |
            test-binaries-

      - name: Build test binaries once
        run: |
          # Compile all tests (reuse across shards)
          go test -c ./... -o .test-cache/
```

**Expected Improvement:**
- First run: 15 min (cold cache)
- Subsequent runs: 8 min (warm cache)
- Cache hit rate: 80-90%

---

#### 2.3 Fail Fast on Linting

**Problem:** CI runs full test suite even if linting fails

**Solution:** Run linting first, fail fast

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: golangci-lint
        uses: golangci/golangci-lint-action@v3
        with:
          version: v1.54

  test:
    needs: lint  # ✅ Only run if lint passes
    runs-on: ubuntu-latest
    # ... test steps
```

---

### 3. Coverage Improvements

#### 3.1 Coverage Gap Analysis

**Tool:** Automated coverage diff

```bash
#!/bin/bash
# tools/coverage-diff.sh

# Generate coverage for PR
go test -coverprofile=coverage-pr.out ./...

# Generate coverage for main
git checkout main
go test -coverprofile=coverage-main.out ./...
git checkout -

# Diff coverage
go run tools/coverage-diff/main.go \
  --before=coverage-main.out \
  --after=coverage-pr.out \
  --threshold=-2.0  # Fail if coverage drops >2%
```

**CI Integration:**
```yaml
- name: Coverage diff
  run: |
    ./tools/coverage-diff.sh
    # Upload to Codecov
    bash <(curl -s https://codecov.io/bash) -f coverage-pr.out
```

---

#### 3.2 Priority Coverage Areas

**Target packages for 80% coverage:**

```
Priority 1 (Critical - must have 85%+):
├─ service/history/workflow/mutable_state_impl.go  (current ~70%, target 85%)
├─ service/history/historyEngine.go                (current ~75%, target 85%)
├─ service/matching/physical_task_queue_manager.go (current ~65%, target 85%)
└─ common/persistence/sql/sqlplugin/               (current ~60%, target 85%)

Priority 2 (Important - target 80%):
├─ service/frontend/handler.go
├─ service/history/replication/
└─ common/namespace/

Priority 3 (Nice to have - target 75%):
├─ service/worker/
└─ common/archiver/
```

**Coverage Test Template:**
```go
// Example: test error paths
func TestMutableState_UpdateWorkflow_PersistenceError(t *testing.T) {
    mockPersistence := &MockPersistence{
        UpdateWorkflowExecutionFunc: func(...) error {
            return errors.New("database unavailable")
        },
    }

    ms := NewMutableState(mockPersistence, ...)

    err := ms.UpdateWorkflow(context.Background(), ...)

    require.Error(t, err)
    require.Contains(t, err.Error(), "database unavailable")
    // Verify state was not corrupted
    require.Equal(t, originalState, ms.GetCurrentState())
}
```

---

### 4. Flaky Test Elimination

#### 4.1 Mandatory Race Detector

**Current:** Race detector only in dedicated job

**Proposed:** Always on

```makefile
# Makefile
test:
	go test -race -short ./...  # ✅ Always with -race

test-integration:
	go test -race -tags=integration ./...  # ✅ Always with -race
```

**Performance Impact:**
- Slower execution (~2x)
- But: catches races early, prevents flaky tests

---

#### 4.2 Goroutine Leak Detection

**Implementation:**
```go
// File: common/testing/leakdetect/leakdetect.go
package leakdetect

import (
    "runtime"
    "testing"
    "time"
)

// CheckGoroutines verifies no goroutines leaked
func CheckGoroutines(t *testing.T) func() {
    before := runtime.NumGoroutine()

    return func() {
        // Wait for goroutines to finish
        time.Sleep(100 * time.Millisecond)

        after := runtime.NumGoroutine()
        if after > before+5 {  // Allow small variance
            t.Errorf("Goroutine leak detected: before=%d after=%d", before, after)

            // Print goroutine stack traces
            buf := make([]byte, 1<<20)
            stackSize := runtime.Stack(buf, true)
            t.Logf("Goroutine stacks:\n%s", buf[:stackSize])
        }
    }
}
```

**Usage:**
```go
func TestSomething(t *testing.T) {
    defer leakdetect.CheckGoroutines(t)()

    // Test code that might leak goroutines
}
```

---

#### 4.3 Proper Cleanup Hooks

**Problem:** Database state leaks between tests

**Solution:** Mandatory cleanup

```go
// File: common/testing/testbase/base.go
type TestBase struct {
    suite.Suite
    cleanupFuncs []func()
}

func (s *TestBase) SetupTest() {
    s.cleanupFuncs = make([]func(), 0)
}

func (s *TestBase) TearDownTest() {
    // Run cleanup in reverse order
    for i := len(s.cleanupFuncs) - 1; i >= 0; i-- {
        s.cleanupFuncs[i]()
    }
}

func (s *TestBase) AddCleanup(f func()) {
    s.cleanupFuncs = append(s.cleanupFuncs, f)
}
```

**Usage:**
```go
func (s *MySuite) TestWorkflow() {
    // Create workflow
    workflowID := "test-workflow-123"
    s.engine.StartWorkflow(...)

    // Register cleanup
    s.AddCleanup(func() {
        s.engine.DeleteWorkflow(workflowID)
    })

    // Test continues... cleanup happens automatically
}
```

---

### 5. Local Testing Experience

#### 5.1 Fast Local Test Command

**New Makefile target:**
```makefile
# Run fast subset of tests locally (< 5 minutes)
test-local:
	@echo "Running fast local test suite..."
	go test -short -race ./service/frontend/... ./service/history/... \
	  -run TestImportant  # Only tests marked as important
	@echo "✅ Fast tests passed! Run 'make test-all' before pushing."

# Run focused tests with one lightweight DB
test-quick:
	docker-compose up -d sqlite
	go test -race ./... -tags=integration -db=sqlite
	docker-compose down
```

#### 5.2 Test Selection by Tags

```go
// Mark critical tests
// +build integration,critical

func TestCriticalWorkflowPath(t *testing.T) {
    // Critical test that must always pass
}
```

```bash
# Run only critical tests
go test -tags=critical ./...
```

---

## Implementation Plan

### Week 1: Test Utilities & Flaky Test Fixes

**Day 1-2: WorkflowBuilder**
```bash
git checkout -b test-infra-improvements

# Create workflow builder
mkdir -p common/testing/workflowbuilder
# Implement builder.go (as shown above)

# Create tests
go test ./common/testing/workflowbuilder/...
```

**Day 3-4: Deterministic Clock**
```bash
# Enhance common/clock with mocking
# Refactor 5-10 flaky time-based tests to use MockedTimeSource

# Example files to fix:
# - service/history/historyEngine2_test.go (lines with time.Sleep)
# - service/matching/physical_task_queue_manager_test.go
```

**Day 5: Cleanup & Leak Detection**
```bash
# Add goroutine leak detection to test base
# Add cleanup hooks to 10 high-value test suites
# Run with -race to verify
```

**Deliverables:**
- ✅ WorkflowBuilder utility (500 LOC)
- ✅ MockedTimeSource (300 LOC)
- ✅ 10 flaky tests fixed
- ✅ Leak detection framework

---

### Week 2: CI Optimization

**Day 1-2: Smart Test Sharding**
```bash
# Create test sharding tool
mkdir -p tools/shard-tests
# Implement timing-based test distribution

# Test locally
go run tools/shard-tests/main.go --shard=1 --total=5
```

**Day 3-4: CI Pipeline Refactor**
```bash
# Update .github/workflows/test.yml
# Add caching layers
# Implement parallel database tests
# Add fail-fast linting
```

**Day 5: Validation**
```bash
# Trigger CI, measure improvements
# Expected: 90+ min → 45 min
```

**Deliverables:**
- ✅ Smart test sharding (200 LOC tool)
- ✅ Optimized CI workflow
- ✅ CI time reduced to <45 min

---

### Week 3: Coverage Improvements

**Day 1: Coverage Analysis**
```bash
# Generate current coverage report
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out -o coverage.html

# Identify gaps with coverage diff tool
go run tools/coverage-diff/main.go --analyze
```

**Day 2-4: Add Missing Tests**

Focus on priority packages:
```bash
# Day 2: service/history/workflow/mutable_state_impl.go
# Add 15-20 new test cases for error paths
# Target: 70% → 85% coverage

# Day 3: service/matching/physical_task_queue_manager.go
# Add concurrent test cases
# Target: 65% → 85% coverage

# Day 4: common/persistence/sql/
# Add error injection tests
# Target: 60% → 85% coverage
```

**Day 5: Validation & Documentation**
```bash
# Run full coverage analysis
# Update documentation
# Create coverage dashboard
```

**Deliverables:**
- ✅ Coverage increased from ~70% to >80%
- ✅ 50-100 new test cases
- ✅ Coverage dashboard

---

## Success Metrics

### Quantitative Metrics

| Metric | Before | Target | Measurement |
|--------|--------|--------|-------------|
| **Test Coverage** | ~70% | >80% | `go test -cover` |
| **CI Duration** | 90-150 min | <45 min | GitHub Actions timing |
| **Flaky Test Rate** | ~10% | <1% | Failed builds / total builds |
| **Local Test Time** | 60+ min | <5 min | `make test-local` timing |
| **Goroutine Leaks** | Unknown | 0 | Automated detection |
| **Race Conditions** | Some | 0 | `-race` detector |

### Qualitative Metrics

- ✅ Developers can run meaningful tests locally in <5 minutes
- ✅ CI failures are always legitimate (not flaky)
- ✅ New contributors can run tests without database setup
- ✅ Coverage reports highlight untested code paths
- ✅ Test utilities reduce boilerplate by 70%

---

## Risks and Mitigations

### Risk 1: Test Refactoring Introduces Bugs

**Likelihood:** Medium
**Impact:** High

**Mitigation:**
- Refactor incrementally (5-10 tests at a time)
- Run both old and new tests in parallel during transition
- Require 2 reviewers for test infrastructure changes
- Monitor CI success rate closely during rollout

---

### Risk 2: CI Optimization Misses Test Failures

**Likelihood:** Low
**Impact:** Critical

**Mitigation:**
- Keep one "full sequential" CI job as sanity check
- Monitor for tests that pass in shards but fail together
- Run full suite weekly on main branch
- Gradual rollout: enable sharding for 10% of PRs first

---

### Risk 3: Developer Pushback on Local Testing

**Likelihood:** Medium
**Impact:** Low

**Mitigation:**
- Make local testing optional but recommended
- Provide clear benefits (faster feedback)
- Document setup clearly
- Offer GitHub Codespaces with pre-configured environment

---

## Alternatives Considered

### Alternative 1: External Test Infrastructure (Buildkite, CircleCI)

**Pros:**
- Better parallelization
- Easier to optimize

**Cons:**
- Additional cost ($500-1000/month)
- Vendor lock-in
- Team must learn new platform

**Decision:** Stick with GitHub Actions for now, revisit if optimization insufficient

---

### Alternative 2: Reduce Test Coverage Requirements

**Pros:**
- Less test maintenance

**Cons:**
- More production bugs
- Lower confidence in changes
- Against industry best practices

**Decision:** Rejected, 80% coverage is reasonable for a project of this importance

---

### Alternative 3: Only Fix Most Flaky Tests

**Pros:**
- Less effort (1 week instead of 3)

**Cons:**
- Doesn't address root causes
- Flakiness will return
- CI remains unreliable

**Decision:** Rejected, systematic fixes are worth the investment

---

## Migration Strategy

### Phase 1: Opt-In (Week 1)

```go
// New utilities available but not required
import "github.com/temporalio/temporal/common/testing/workflowbuilder"

// Developers can use old or new style
```

**Rollout:**
- Announce in eng-all channel
- Document in CONTRIBUTING.md
- Use in 5-10 new tests as examples

---

### Phase 2: Gradual Adoption (Week 2-3)

- Refactor high-value test suites to new utilities
- CI optimization enabled for 50% of PRs (A/B test)
- Coverage requirements enforced for new code only

---

### Phase 3: Full Rollout (Week 4+)

- All new tests must use new utilities
- CI optimization enabled for all PRs
- Coverage requirements enforced for all changes
- Legacy tests refactored opportunistically

---

## Open Questions

1. **Q:** Should we require 80% coverage for all packages or weighted by criticality?
   **A:** Weighted by criticality (see Priority Coverage Areas)

2. **Q:** How to handle tests that genuinely need real time (e.g., performance tests)?
   **A:** Tag with `// +build realtime` and run separately

3. **Q:** Should we auto-retry flaky tests in CI?
   **A:** No, fix the root cause. Retries hide problems.

4. **Q:** Coverage threshold for blocking merges?
   **A:** Don't decrease coverage (ratcheting). Target +1% per quarter.

---

## Related Work

- **RFC-0007: Code Modularization** - Smaller files are easier to test
- **RFC-0003: Developer Experience** - Better local setup improves testing
- **Blog Post 3: Design Patterns** - Testing patterns discussion

---

## References

### Internal
- [Test Coverage Report](https://app.codecov.io/gh/temporalio/temporal)
- [Current CI Configuration](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/.github/workflows/test.yml)
- [Test Utilities](https://github.com/temporalio/temporal/tree/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/common/testing)

### External
- [Google Testing Blog: Test Flakiness](https://testing.googleblog.com/2016/05/flaky-tests-at-google-and-how-we.html)
- [Go Test Coverage](https://go.dev/blog/cover)
- [Deterministic Testing](https://www.sqlite.org/testing.html)

---

**Next Steps:**
1. Review and approve RFC
2. Create GitHub project for tracking
3. Assign owners for each week
4. Begin Week 1 implementation

**Status:** Ready for review
**Estimated Completion:** 3 weeks from approval
