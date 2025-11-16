# RFC-0007: Code Modularization

**Status:** Draft | **Effort:** 6-8 weeks | **Impact:** High
**Author:** Analysis Team | **Created:** 2025-11-16

## Summary

Refactor large files and packages to improve maintainability and reduce complexity.

## Motivation

Large files (>3K LOC) hard to maintain. historyEngine.go (3,500 lines), mutable_state_impl.go (4,000 lines). High complexity increases bug risk.

## Proposed Solution

**Refactoring Strategy:**
1. Split historyEngine.go → workflow_engine.go, activity_engine.go, timer_engine.go
2. Extract interfaces for testability
3. Reduce cyclomatic complexity (target: <15 per function)
4. Create sub-packages in common/persistence
**Approach:** Gradual refactoring, maintain backward compatibility

## Implementation Plan

**Phase 1 (2 weeks):** Split historyEngine.go
**Phase 2 (2 weeks):** Refactor mutable_state_impl.go
**Phase 3 (2 weeks):** Extract persistence sub-packages
**Phase 4 (2 weeks):** Testing, validation
**Total:** 8 weeks

## Success Metrics

- ✅ No files > 2,000 lines
- ✅ Average cyclomatic complexity < 15
- ✅ Zero regressions from refactoring
- ✅ Code review time reduced by 30%

## Risks

**High:** Refactoring could introduce regressions. Mitigation: extensive testing, gradual rollout, feature flags.

---

**Next:** See [Prioritization Matrix](00-prioritization-matrix.md)
