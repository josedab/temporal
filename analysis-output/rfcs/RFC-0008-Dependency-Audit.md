# RFC-0008: Dependency Audit

**Status:** Draft | **Effort:** 4-6 weeks | **Impact:** Medium
**Author:** Analysis Team | **Created:** 2025-11-16

## Summary

Audit dependencies for security vulnerabilities, licensing issues, and maintainability.

## Motivation

79 direct dependencies + 94 transitive. Some unmaintained (lib/pq), potential vulnerabilities, license compliance risks. Need audit and reduction strategy.

## Proposed Solution

**Audit Process:**
1. List all dependencies with: version, last update, maintainer, license
2. Check for CVEs (govulncheck)
3. Identify unmaintained deps (no commits in 12+ months)
4. Find alternatives for problematic deps
5. Create migration plan
**Targets:** lib/pq → pgx/v5, review Elasticsearch client

## Implementation Plan

**Week 1-2:** Dependency audit, CVE scanning
**Week 3-4:** Identify alternatives, create migration plan
**Week 5-8:** Gradual migrations (lib/pq, others)
**Week 9-10:** Testing, validation
**Total:** 10 weeks

## Success Metrics

- ✅ Zero critical vulnerabilities
- ✅ All dependencies updated within 30 days of release
- ✅ License compliance verified
- ✅ Dependency count reduced by 10%

## Risks

**Medium:** Dependency changes could break compatibility. Mitigation: gradual migration, thorough testing.

---

**Next:** See [Prioritization Matrix](00-prioritization-matrix.md)
