# RFC-0006: Documentation Improvements

**Status:** Draft | **Effort:** 2 weeks | **Impact:** Medium
**Author:** Analysis Team | **Created:** 2025-11-16

## Summary

Create operator runbooks, troubleshooting guides, and architecture deep-dives.

## Motivation

Documentation gaps: no operator runbooks, troubleshooting is trial-and-error, advanced features under-documented. Contributors need architecture deep-dives.

## Proposed Solution

**Documentation Types:**
1. Operator Runbooks: deployment, upgrades, disaster recovery, troubleshooting
2. Architecture Deep-Dives: sharding internals, event sourcing, task processing
3. Contributor Guides: code walkthrough, design patterns, testing
4. Troubleshooting: common issues, debugging tips, performance tuning
**Format:** Markdown in /docs, published to docs.temporal.io

## Implementation Plan

**Week 1:** Operator runbooks (deployment, upgrades, DR)
**Week 2:** Architecture deep-dives (sharding, event sourcing, queues)
**Week 3:** Troubleshooting guide, performance tuning
**Week 4:** Review, feedback, publishing
**Total:** 4 weeks

## Success Metrics

- ✅ All runbooks completed and published
- ✅ Contributor ramp-up time reduced by 50%
- ✅ Documentation rated 4+/5 by users
- ✅ 50% reduction in Slack support questions

## Risks

**Low:** Documentation only, no code changes. Risk: docs become outdated. Mitigation: review process, automated checks.

---

**Next:** See [Prioritization Matrix](00-prioritization-matrix.md)
