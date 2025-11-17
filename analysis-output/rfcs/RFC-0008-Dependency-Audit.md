# RFC-0008: Dependency Audit and Security Hardening

**Status:** Draft
**Effort:** 4-6 weeks
**Impact:** Medium (Security Critical)
**Author:** Analysis Team
**Created:** 2025-11-16
**Commit SHA:** `0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3`

---

## Summary

Perform comprehensive audit of all 173 dependencies (79 direct, 94 indirect) for security vulnerabilities, licensing issues, and maintainability concerns. Migrate from unmaintained dependencies (`lib/pq` in maintenance mode), establish automated CVE monitoring, and reduce dependency count by 10%. This ensures long-term security, compliance, and maintainability of the Temporal server.

---

## Motivation

### Current Dependency State

**Total Dependencies:**
- **Direct:** 79
- **Indirect:** 94
- **Total:** 173

**By Category:**
| Category | Count | Examples |
|----------|-------|----------|
| Infrastructure | 25 | gRPC, database drivers, Ringpop |
| Observability | 15 | Tally, OpenTelemetry, Prometheus |
| Utilities | 25 | UUID, cron, time, codec |
| Testing | 10 | Testify, mock |
| Security | 4 | JWT, JOSE, crypto |

---

### Problem 1: Unmaintained Dependencies

#### lib/pq (PostgreSQL Driver)

**Status:** Maintenance mode since 2021
**Current Version:** 1.10.9
**Last Significant Update:** 2021

**Issues:**
- No active feature development
- Security patches only
- Community recommends migrating to `pgx`

**Impact:**
```
Risk Level: Medium
├─ Security: CVEs may not be patched quickly
├─ Features: Missing modern PostgreSQL features
└─ Performance: pgx is 2-3x faster for many operations
```

**Migration Target:** `github.com/jackc/pgx/v5` (already used, but lib/pq still present)

---

#### gocql (Cassandra Driver)

**Status:** Community-maintained, not official DataStax driver
**Current Version:** 1.7.0

**Issues:**
- Not the official driver
- Slower response to issues
- Limited enterprise support

**Impact:**
```
Risk Level: Low-Medium
├─ Security: Community support is good
├─ Features: Mostly feature-complete
└─ Support: No official backing
```

**Decision:** Continue monitoring, acceptable for now

---

#### olivere/elastic (Elasticsearch Client)

**Status:** Community-maintained, not official Elastic client
**Current Version:** v7.0.32 (for Elasticsearch 7.x)

**Issues:**
- Not the official Elastic client
- Elasticsearch 8.x migration will be challenging
- API compatibility concerns

**Impact:**
```
Risk Level: Medium
├─ Security: Community support decent
├─ Compatibility: Elasticsearch 8.x requires review
└─ Future: May need official client
```

**Decision:** Evaluate official Elastic client for ES 8.x migration

---

### Problem 2: Security Vulnerabilities

**Current CVE Status** (as of 2025-11-16):

```bash
# Run vulnerability check
go run golang.org/x/vuln/cmd/govulncheck@latest ./...
```

**Known Issues:**
- **No critical CVEs currently** (good!)
- **3-4 low/medium advisories** in transitive dependencies
- **Risk:** New CVEs discovered regularly

**Concern:**
- No automated monitoring
- Manual checks infrequent
- Response time: days to weeks

**Target:**
- Automated daily CVE scanning
- Response time: <24 hours for critical, <1 week for medium

---

### Problem 3: License Compliance

**License Distribution:**
| License | Percentage | Risk Level |
|---------|------------|------------|
| MIT | 60% | ✅ Low (permissive) |
| Apache 2.0 | 30% | ✅ Low (permissive) |
| BSD-3-Clause | 8% | ✅ Low (permissive) |
| BSD-2-Clause | 2% | ✅ Low (permissive) |

**Good News:** No GPL/AGPL dependencies (copyleft licenses avoided)

**Concerns:**
- No automated license checking
- Manual audit is time-consuming
- New dependencies might introduce problematic licenses

**Target:**
- Automated license scanning in CI
- Allowlist/blocklist enforcement

---

### Problem 4: Dependency Bloat

**Examples of Overlapping Dependencies:**

1. **Multiple PostgreSQL Drivers:**
   ```
   github.com/jackc/pgx/v5           # Modern, fast
   github.com/lib/pq                 # Maintenance mode
   ```
   **Why both?** Legacy code + migration in progress

2. **Multiple Mock Libraries:**
   ```
   github.com/golang/mock    # Original gomock (archived)
   go.uber.org/mock          # Uber fork (active)
   ```
   **Why both?** Migration in progress

3. **Multiple Time Libraries:**
   ```
   golang.org/x/time         # Official rate limiting
   github.com/robfig/cron/v3 # Cron parsing
   ```
   **Acceptable:** Different use cases

**Opportunity:** Remove fully migrated dependencies

---

### Problem 5: Dependency Update Lag

**Update Frequency:**
```
Immediate (<1 week):     20% of dependencies
Fast (1-4 weeks):        40% of dependencies
Slow (1-3 months):       30% of dependencies
Very Slow (3-6 months):  10% of dependencies
```

**Examples of Lag:**
- gRPC: Usually 2-4 weeks behind latest
- Protobuf: Usually 1-2 weeks behind latest
- Database drivers: Usually 1-2 months behind

**Why Lag Occurs:**
- Manual update process
- Fear of breaking changes
- Testing overhead

**Target:** 80% of dependencies updated within 30 days

---

## Proposed Solution

### Solution Architecture

```
┌─────────────────────────────────────────────────────┐
│       Dependency Management Strategy                │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Phase 1: Audit & Assessment                        │
│  ├─ Catalog all dependencies                        │
│  ├─ CVE scanning                                    │
│  ├─ License compliance check                        │
│  └─ Maintainability assessment                      │
│                                                     │
│  Phase 2: Automated Monitoring                      │
│  ├─ Dependabot / Renovate setup                     │
│  ├─ Daily CVE scanning (GitHub Actions)            │
│  ├─ License scanning (licensed tool)               │
│  └─ Dependency dashboard                            │
│                                                     │
│  Phase 3: Strategic Migrations                      │
│  ├─ lib/pq → full pgx/v5 migration                 │
│  ├─ golang/mock → go.uber.org/mock                 │
│  ├─ Evaluate Elasticsearch 8.x client              │
│  └─ Remove unused dependencies                      │
│                                                     │
│  Phase 4: Dependency Policies                       │
│  ├─ Approval process for new dependencies          │
│  ├─ Regular security reviews                        │
│  └─ Deprecated dependency sunset policy            │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## Detailed Design

### 1. Comprehensive Dependency Audit

#### 1.1 Dependency Catalog

**Tool:** Create automated catalog

```bash
#!/bin/bash
# tools/dependency-audit.sh

echo "Dependency Audit Report"
echo "Generated: $(date)"
echo "========================"

# List all dependencies with metadata
go list -m -json all | jq -r '
  select(.Path) |
  "\(.Path)
,\(.Version),\(.Time // "unknown")"
' | while IFS=, read -r path version time; do
  # Check last commit
  echo "$path,$version,$time"

  # Check for CVEs
  govulncheck -mode=query "$path" 2>/dev/null || echo "  No CVEs"

  # Check license
  go-licenses report "$path" 2>/dev/null || echo "  License: Unknown"
done
```

**Output Format:**
```csv
Package,Version,Last Update,CVEs,License,Maintainer Status
github.com/lib/pq,v1.10.9,2023-03-15,0,MIT,Maintenance Mode
github.com/jackc/pgx/v5,v5.7.2,2025-01-10,0,MIT,Active
...
```

---

#### 1.2 CVE Scanning

**Daily Automated Scan:**

```yaml
# .github/workflows/security-scan.yml
name: Security Scan

on:
  schedule:
    - cron: '0 6 * * *'  # Daily at 6 AM UTC
  push:
    branches: [main]

jobs:
  vulnerability-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.25'

      - name: Run govulncheck
        run: |
          go install golang.org/x/vuln/cmd/govulncheck@latest
          govulncheck -test ./...

      - name: Report vulnerabilities
        if: failure()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "🚨 Security vulnerabilities detected in dependencies!",
              "channel": "#security-alerts"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

---

#### 1.3 License Compliance

**Automated License Check:**

```yaml
# .github/workflows/license-check.yml
name: License Compliance

on: [push, pull_request]

jobs:
  check-licenses:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Check licenses
        uses: google/go-licenses@v1
        with:
          mode: check
          allowed-licenses: MIT,Apache-2.0,BSD-3-Clause,BSD-2-Clause,ISC
          disallowed-licenses: GPL,LGPL,AGPL,SSPL,CC-BY-NC

      - name: Generate license report
        run: |
          go-licenses report ./... > licenses.csv
          cat licenses.csv

      - name: Upload license report
        uses: actions/upload-artifact@v3
        with:
          name: licenses
          path: licenses.csv
```

---

### 2. Automated Dependency Updates

#### 2.1 Renovate Bot Configuration

**Install Renovate:**

```json
// renovate.json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:base"],
  "schedule": ["every weekend"],
  "packageRules": [
    {
      "matchUpdateTypes": ["patch"],
      "automerge": true,
      "automergeType": "pr",
      "requiredStatusChecks": ["test", "lint"]
    },
    {
      "matchUpdateTypes": ["minor"],
      "groupName": "minor dependencies",
      "schedule": ["every 2 weeks"]
    },
    {
      "matchUpdateTypes": ["major"],
      "groupName": "major dependencies",
      "schedule": ["every month"],
      "reviewers": ["team:temporal-infrastructure"]
    },
    {
      "matchPackagePatterns": ["^google.golang.org/grpc"],
      "schedule": ["every week"],
      "reviewers": ["team:temporal-core"]
    },
    {
      "matchPackagePatterns": ["^github.com/jackc/pgx"],
      "schedule": ["every week"]
    }
  ],
  "vulnerabilityAlerts": {
    "labels": ["security"],
    "assignees": ["security-team"],
    "enabled": true
  }
}
```

**Benefits:**
- Automated PRs for dependency updates
- Security alerts elevated
- Major updates require review
- Patch updates auto-merge

---

#### 2.2 Dependency Update Policy

**Decision Matrix:**

| Update Type | Auto-Merge | Review Required | Schedule |
|-------------|------------|-----------------|----------|
| **Security Patch** | ✅ Yes (after CI) | No | Immediate |
| **Patch (x.y.Z)** | ✅ Yes (after CI) | No | Weekly |
| **Minor (x.Y.z)** | ❌ No | 1 reviewer | Bi-weekly |
| **Major (X.y.z)** | ❌ No | 2 reviewers + RFC | Monthly |

---

### 3. Strategic Dependency Migrations

#### 3.1 lib/pq → pgx/v5 Complete Migration

**Current State:**
```go
// Some code uses lib/pq
import "github.com/lib/pq"

// Other code uses pgx/v5
import "github.com/jackc/pgx/v5"
```

**Migration Plan:**

**Step 1: Find all lib/pq usages**
```bash
# Search for lib/pq imports
grep -r "github.com/lib/pq" --include="*.go" .

# Example results:
# common/persistence/sql/sqlplugin/postgresql/plugin.go
# tools/sql/some-tool.go
```

**Step 2: Replace lib/pq with pgx/v5**

Before:
```go
// common/persistence/sql/sqlplugin/postgresql/plugin.go
import (
    "database/sql"
    _ "github.com/lib/pq"
)

func NewPlugin() *Plugin {
    db, err := sql.Open("postgres", connectionString)
    // ...
}
```

After:
```go
// common/persistence/sql/sqlplugin/postgresql/plugin.go
import (
    "github.com/jackc/pgx/v5"
    "github.com/jackc/pgx/v5/stdlib"  // database/sql compatibility
)

func NewPlugin() *Plugin {
    // Register pgx driver
    stdlib.RegisterConnConfig(config)
    db, err := sql.Open("pgx", connectionString)
    // ...
}
```

**Step 3: Update tools**

```bash
# tools/sql may use lib/pq directly
# Refactor to use pgx/v5
```

**Step 4: Remove lib/pq from go.mod**

```bash
go mod tidy
# Verify lib/pq is gone
go list -m all | grep lib/pq
# Should return nothing
```

**Expected Impact:**
- **Performance:** 2-3x faster queries (pgx is more efficient)
- **Features:** Access to PostgreSQL-specific features
- **Security:** Active maintenance, faster CVE patches

---

#### 3.2 golang/mock → go.uber.org/mock Migration

**Status:** Partially migrated

**Remaining Work:**
```bash
# Find remaining golang/mock usage
grep -r "github.com/golang/mock" --include="*.go" .

# Example:
# service/history/some_test.go
# service/matching/some_test.go
```

**Migration:**

Before:
```go
//go:generate mockgen -source=interface.go -package=mocks -destination=mocks/mock_interface.go

import "github.com/golang/mock/gomock"

func TestSomething(t *testing.T) {
    ctrl := gomock.NewController(t)
    defer ctrl.Finish()
    // ...
}
```

After:
```go
//go:generate mockgen -source=interface.go -package=mocks -destination=mocks/mock_interface.go

import "go.uber.org/mock/gomock"

func TestSomething(t *testing.T) {
    ctrl := gomock.NewController(t)
    defer ctrl.Finish()
    // ...
}
```

**Automation:**
```bash
# Use sed to replace imports
find . -name "*.go" -type f -exec sed -i 's|github.com/golang/mock|go.uber.org/mock|g' {} +

# Regenerate mocks
go generate ./...

# Test
go test ./...
```

---

#### 3.3 Elasticsearch Client Evaluation

**Current:** `github.com/olivere/elastic/v7` (community client for ES 7.x)

**Options for ES 8.x:**

**Option 1: Stick with olivere/elastic**
```go
import "github.com/olivere/elastic/v8"  // When available
```
**Pros:**
- Familiar API
- Minimal code changes

**Cons:**
- Community-maintained
- Slower updates

**Option 2: Official Elastic Client**
```go
import "github.com/elastic/go-elasticsearch/v8"
```
**Pros:**
- Official support
- Latest features
- Better documentation

**Cons:**
- Different API (requires rewrite)
- Migration effort: 2-3 weeks

**Recommendation:** Evaluate both when ES 8.x migration occurs. Prefer official client if API differences are manageable.

---

### 4. Dependency Reduction

#### 4.1 Remove Unused Dependencies

**Candidates for Removal:**

```bash
# Use go mod tidy to find unused
go mod tidy

# Use modgraphviz to visualize
go mod graph | modgraphviz | dot -Tsvg -o dependencies.svg

# Manual check for test-only dependencies in main code
go list -deps ./... | grep "test"
```

**Potential Removals:**
- Old testing utilities no longer used
- Replaced dependencies (after migrations complete)

---

#### 4.2 Consolidate Similar Dependencies

**Example: Time Utilities**

Current:
```go
import (
    "golang.org/x/time/rate"     // Rate limiting
    "github.com/robfig/cron/v3"  // Cron parsing
)
```

**Analysis:** Both needed for different purposes, cannot consolidate.

---

### 5. Dependency Approval Process

#### 5.1 New Dependency Checklist

**Before adding a new dependency, verify:**

```markdown
## New Dependency Checklist

- [ ] **Necessity**: Cannot be implemented reasonably in-house?
- [ ] **Maintenance**: Active commits in last 6 months?
- [ ] **Popularity**: 1000+ GitHub stars OR official library?
- [ ] **License**: MIT, Apache 2.0, or BSD?
- [ ] **Security**: No known CVEs?
- [ ] **Size**: <10 transitive dependencies?
- [ ] **Documentation**: Good docs and examples?
- [ ] **Alternatives**: Compared to 2+ alternatives?
- [ ] **Team Approval**: 2+ thumbs up in RFC/PR?
```

---

#### 5.2 Dependency RFC Template

```markdown
# RFC: Add Dependency [Package Name]

## Motivation
Why do we need this dependency?

## Alternatives Considered
1. **Implement in-house**: [Effort estimate]
2. **Alternative package 1**: [Pros/cons]
3. **Alternative package 2**: [Pros/cons]

## Chosen Solution: [Package Name]
- **Version**: vX.Y.Z
- **License**: MIT
- **Maintainer**: [GitHub org/person]
- **Stars**: 5,000+
- **Last Update**: 2025-01-15
- **CVEs**: None
- **Transitive Deps**: 3

## Impact
- Binary size impact: +500KB
- Build time impact: +10s
- Maintenance burden: Low

## Decision
Approved by: @alice, @bob, @charlie
Date: 2025-01-20
```

---

## Implementation Plan

### Phase 1: Audit & Setup (Weeks 1-2)

**Week 1: Dependency Catalog**
```bash
# Day 1-2: Generate dependency report
./tools/dependency-audit.sh > dependency-report.csv

# Day 3: CVE scanning
go install golang.org/x/vuln/cmd/govulncheck@latest
govulncheck -test ./...

# Day 4: License scanning
go install github.com/google/go-licenses@latest
go-licenses report ./... > licenses.csv

# Day 5: Analysis and prioritization
# Identify critical issues
```

**Week 2: Automated Monitoring Setup**
```bash
# Day 1-2: Setup Renovate Bot
# Add renovate.json to repo
# Configure GitHub integration

# Day 3: Setup security scanning
# Add .github/workflows/security-scan.yml
# Configure Slack alerts

# Day 4: Setup license checking
# Add .github/workflows/license-check.yml
# Configure allowlist/blocklist

# Day 5: Dashboard creation
# Create dependency health dashboard (Grafana or similar)
```

**Deliverables:**
- ✅ Complete dependency catalog
- ✅ CVE scan results (0 critical, document any medium/low)
- ✅ License compliance report
- ✅ Automated monitoring active

---

### Phase 2: Critical Migrations (Weeks 3-5)

**Week 3: lib/pq Migration**
```bash
# Day 1: Find all usages
grep -r "github.com/lib/pq" --include="*.go" .

# Day 2-3: Replace with pgx/v5
# Update imports, test each file

# Day 4: Integration testing
make test-integration-postgresql

# Day 5: Performance validation
# Run benchmarks, compare before/after
```

**Week 4: golang/mock Migration**
```bash
# Day 1: Find remaining usages
grep -r "github.com/golang/mock" --include="*.go" .

# Day 2: Automated replacement
find . -name "*.go" -exec sed -i 's|github.com/golang/mock|go.uber.org/mock|g' {} +

# Day 3: Regenerate mocks
go generate ./...

# Day 4-5: Test and validate
make test
```

**Week 5: Dependency Removal**
```bash
# Day 1: Remove fully migrated dependencies
go mod tidy

# Day 2-3: Verify no breakage
make test-all

# Day 4: Update documentation
# Update CONTRIBUTING.md with new dependency guidelines

# Day 5: Code review and merge
```

**Deliverables:**
- ✅ lib/pq completely removed
- ✅ golang/mock completely removed
- ✅ All tests pass
- ✅ Performance benchmarks stable or improved

---

### Phase 3: Policy & Governance (Week 6)

**Week 6: Establish Policies**
```bash
# Day 1-2: Document dependency policies
# Create docs/dependencies/POLICY.md

# Day 3: Create RFC template for new dependencies
# Add to .github/PULL_REQUEST_TEMPLATE/dependency.md

# Day 4: Team training
# Internal presentation on dependency management

# Day 5: Launch dependency review board
# Assign rotating reviewers
```

**Deliverables:**
- ✅ Dependency policy documented
- ✅ RFC template created
- ✅ Team trained
- ✅ Review process established

---

## Success Metrics

### Quantitative Metrics

| Metric | Baseline | Target | Measurement |
|--------|----------|--------|-------------|
| **Critical CVEs** | 0 | 0 | `govulncheck` |
| **Medium/Low CVEs** | 3-4 | 0 | `govulncheck` |
| **Dependencies in Maintenance Mode** | 2 | 0 | Manual review |
| **Dependency Count** | 173 | 155 (-10%) | `go list -m all | wc -l` |
| **Update Lag (Median)** | 45 days | 15 days | Renovate dashboard |
| **License Compliance** | 100% | 100% | `go-licenses check` |
| **Time to Patch Critical CVE** | 7 days | 1 day | Incident tracking |

### Qualitative Metrics

**Security Posture:**
- ✅ Automated CVE monitoring active
- ✅ Response playbook defined
- ✅ All dependencies from trusted sources

**Developer Experience:**
- ✅ Clear process for adding dependencies
- ✅ Automated updates reduce manual work
- ✅ Dependency health visible (dashboard)

---

## Risks and Mitigations

### Risk 1: Breaking Changes from Updates

**Likelihood:** Medium
**Impact:** High

**Mitigation:**
- Comprehensive test suite (RFC-0005)
- Canary deployments for dependency updates
- Automated rollback on test failures
- Version pinning for critical dependencies

---

### Risk 2: Maintenance Burden of Monitoring

**Likelihood:** Low
**Impact:** Medium

**Mitigation:**
- Automation reduces manual work
- Weekly 15-minute dependency review meeting
- Rotating responsibility among team
- Dashboard for quick health check

---

### Risk 3: False Positives from Scanners

**Likelihood:** Medium
**Impact:** Low

**Mitigation:**
- Triaging process for alerts
- Allowlist for known false positives
- Regular review of suppressed alerts
- Human review before taking action

---

## Alternatives Considered

### Alternative 1: No Automated Updates

**Approach:** Manual dependency updates only

**Pros:**
- Full control over updates
- No surprise breakage

**Cons:**
- **Slow response to CVEs** (days/weeks)
- High manual effort
- Update lag accumulates

**Decision:** Rejected, automation is essential for security

---

### Alternative 2: Auto-Merge All Updates

**Approach:** Automatically merge all updates after CI passes

**Pros:**
- Always up to date
- Minimal manual work

**Cons:**
- **High risk of breakage**
- Major version changes need review
- Violates change control policies

**Decision:** Rejected, balanced approach better (patch auto-merge, major manual)

---

### Alternative 3: Vendoring Dependencies

**Approach:** Use `go mod vendor` to commit dependencies

**Pros:**
- Reproducible builds
- No external network dependencies

**Cons:**
- **Large repository size** (10x increase)
- Slower clones
- Harder to see dependency changes in PRs
- Go modules already provide reproducibility

**Decision:** Rejected, Go modules sufficient

---

## Migration Strategy

### Phase 0: Communication (Before Week 1)

**Announcement:**
```
Subject: Dependency Audit and Security Hardening Initiative

Team,

We're launching a 6-week initiative to improve our dependency management:

1. Audit all 173 dependencies for security/licensing
2. Setup automated CVE monitoring
3. Migrate from unmaintained deps (lib/pq → pgx/v5)
4. Establish governance policies

Why:
- Reduce security risk (faster CVE response)
- Improve maintainability (active dependencies only)
- Better compliance (license tracking)

Impact to you:
- New dependency approval process (RFC required)
- Automated PRs from Renovate Bot (review/approve)
- Weekly dependency review meeting (15 min, rotating)

Questions? Ask in #dependency-audit

Thanks,
Infrastructure Team
```

---

### Phase 1-3: Gradual Rollout (Weeks 1-6)

**Weekly Rhythm:**
1. Monday: Plan week's work
2. Tue-Thu: Execute migrations/setup
3. Friday: Review and merge
4. Weekend: Monitor for issues

**Rollback Plan:**
- Keep old dependencies in go.mod for 1 week after migration
- Revert PRs if issues detected
- Document rollback procedures

---

### Phase 4: Steady State (Week 7+)

**Ongoing Operations:**
- **Daily:** Automated CVE scans
- **Weekly:** Dependency health review (15 min)
- **Monthly:** Dependency update batch review
- **Quarterly:** Full dependency audit

---

## Open Questions

1. **Q:** Should we pin exact versions or use ranges?
   **A:** Pin exact versions in go.mod (default behavior). Ranges too risky.

2. **Q:** Who reviews major version updates?
   **A:** Infrastructure team + service owner (2 reviewers minimum)

3. **Q:** How to handle transitive dependency CVEs?
   **A:** Update direct dependency to version that includes fix. If not possible, fork or find alternative.

4. **Q:** Elasticsearch 8 migration timeline?
   **A:** Not urgent, ES 7 supported until 2026. Evaluate in 2025 Q3.

---

## Related Work

- **RFC-0009: Security Hardening** - Complements dependency security
- **RFC-0005: Testing Infrastructure** - Better tests enable safer updates
- **Blog Post 1: Architecture** - Dependency decisions explained

---

## References

### Tools
- [govulncheck](https://pkg.go.dev/golang.org/x/vuln/cmd/govulncheck) - Go vulnerability scanner
- [go-licenses](https://github.com/google/go-licenses) - License checker
- [Renovate](https://docs.renovatebot.com/) - Automated dependency updates
- [Dependabot](https://github.com/dependabot) - GitHub's dependency updater

### Security Resources
- [National Vulnerability Database](https://nvd.nist.gov/)
- [Go Security Policy](https://go.dev/security)
- [OWASP Dependency Check](https://owasp.org/www-project-dependency-check/)

### Temporal Resources
- [Current Dependencies](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/go.mod)
- [Dependency Graph Analysis](../initial-analysis/dependency-graph.md)

---

**Next Steps:**
1. Review and approve RFC
2. Create dependency audit project
3. Setup monitoring (Week 1)
4. Begin migrations (Week 3)

**Status:** Ready for review
**Estimated Completion:** 6 weeks from approval
