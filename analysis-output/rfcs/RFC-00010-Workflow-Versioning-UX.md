# RFC-00010: Workflow Versioning UX

**Status:** Draft | **Effort:** 2-3 weeks | **Impact:** Low-Medium
**Author:** Analysis Team | **Created:** 2025-11-16

## Summary

Improve UX for worker versioning and deployment management.

## Motivation

Worker versioning UX is complex. Operators struggle with deployment rules, build IDs, version sets. Need simpler mental model and better tooling.

## Proposed Solution

**UX Improvements:**
1. Simplified CLI: temporal deployment create, temporal deployment rollout
2. Automatic build ID: infer from git SHA + timestamp
3. Deployment wizard: interactive setup for deployment rules
4. Better error messages: explain why task routed to specific build ID
5. Deployment history: visualize rollout progress
**Goal:** Deployment in 3 commands instead of 10+

## Implementation Plan

**Week 1:** CLI improvements, simplified commands
**Week 2:** Deployment wizard, better UX
**Week 3:** Testing, documentation
**Total:** 3 weeks

## Success Metrics

- ✅ Deployment setup < 3 commands
- ✅ User satisfaction 4+/5
- ✅ 50% reduction in deployment-related issues
- ✅ Adoption of worker versioning increased 2x

## Risks

**Low:** UX improvements, backward compatible. Risk: users dislike new UX. Mitigation: user testing, feedback.

---

**Next:** See [Prioritization Matrix](00-prioritization-matrix.md)
