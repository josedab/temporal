# RFC-0005: Testing Infrastructure

**Status:** Draft | **Effort:** 2-3 weeks | **Impact:** High
**Author:** Analysis Team | **Created:** 2025-11-16

## Summary

Enhance testing infrastructure with better coverage, faster CI, and reduced flakiness.

## Motivation

Current testing gaps: coverage ~70%, CI takes 90+ minutes, flaky tests cause 10% failure rate. Need better test utilities, faster CI, comprehensive coverage.

## Proposed Solution

**Improvements:**
1. Test utilities: more test helpers (TaskPoller enhancements, deterministic clocks)
2. CI optimization: parallel execution, better caching, shard tests
3. Coverage: target 80%+ coverage, missing areas identified
4. Flakiness: retry only truly flaky tests, fix root causes
**Goal:** CI < 45 minutes, flakiness < 1%

## Implementation Plan

**Week 1:** Enhance test utilities, fix flaky tests
**Week 2:** CI optimization (parallel, caching)
**Week 3:** Coverage analysis, add missing tests
**Total:** 3 weeks

## Success Metrics

- ✅ Test coverage > 80%
- ✅ CI time < 45 minutes
- ✅ Flaky test rate < 1%
- ✅ Zero failed releases due to tests

## Risks

**Medium:** Test changes could introduce new bugs. Mitigation: gradual rollout, extensive validation.

---

**Next:** See [Prioritization Matrix](00-prioritization-matrix.md)
