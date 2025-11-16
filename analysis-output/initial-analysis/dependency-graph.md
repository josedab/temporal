# Temporal Server Dependency Analysis

**Analysis Date:** 2025-11-16
**Commit SHA:** `0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3`

---

## Overview

This document analyzes the dependency structure of the Temporal server, including both external dependencies and internal module relationships.

---

## External Dependencies Summary

### Dependency Statistics

| Metric | Count |
|--------|-------|
| **Direct Dependencies** | 79 |
| **Indirect Dependencies** | 94 |
| **Total Dependencies** | 173 |
| **Go Version** | 1.25.0 |

---

## Critical External Dependencies

### Core Infrastructure

#### **gRPC & Protocol Buffers**
| Dependency | Version | Purpose | Last Update |
|------------|---------|---------|-------------|
| `google.golang.org/grpc` | 1.72.2 | RPC framework | Recent (2024+) |
| `google.golang.org/protobuf` | 1.36.6 | Protocol buffers | Recent (2024+) |
| `github.com/grpc-ecosystem/grpc-gateway/v2` | 2.26.1 | HTTP/REST gateway | Recent |

**Risk Assessment:** ✅ **Low** - Well-maintained, core to ecosystem
**Dependencies:** 5 direct, ~20 transitive

---

#### **Dependency Injection**
| Dependency | Version | Purpose | Assessment |
|------------|---------|---------|------------|
| `go.uber.org/fx` | 1.24.0 | Dependency injection | ✅ Active |
| `go.uber.org/dig` | 1.19.0 | DI foundation (fx dependency) | ✅ Active |

**Why Fx?**
- Declarative dependency graph
- Lifecycle management (startup/shutdown)
- Error propagation
- Widely adopted in Go ecosystem

**Trade-offs:**
- ✅ Explicit dependencies, testable
- ❌ Learning curve, reflection overhead

---

### Database Drivers

#### **Cassandra**
| Dependency | Version | Purpose | Assessment |
|------------|---------|---------|------------|
| `github.com/gocql/gocql` | 1.7.0 | Cassandra driver | ⚠️ Community-maintained |

**Status:** Active but not official DataStax driver
**Alternatives:** DataStax driver exists but gocql is more mature
**Risk:** Medium - community support, but well-established

---

#### **PostgreSQL**
| Dependency | Version | Purpose | Assessment |
|------------|---------|---------|------------|
| `github.com/jackc/pgx/v5` | 5.7.2 | PostgreSQL driver | ✅ Active, modern |
| `github.com/lib/pq` | 1.10.9 | Alternate PostgreSQL driver | ⚠️ Maintenance mode |

**Primary:** pgx/v5 (high-performance, actively maintained)
**Secondary:** lib/pq (legacy, used by some tools)
**Recommendation:** Migrate fully to pgx/v5

---

#### **MySQL**
| Dependency | Version | Purpose | Assessment |
|------------|---------|---------|------------|
| `github.com/go-sql-driver/mysql` | 1.9.0 | MySQL driver | ✅ Official, active |

**Status:** Official MySQL driver, well-maintained
**Risk:** Low

---

#### **SQLite**
| Dependency | Version | Purpose | Assessment |
|------------|---------|---------|------------|
| `modernc.org/sqlite` | 1.39.1 | Pure Go SQLite | ✅ Active |

**Why Pure Go?** CGO-free, cross-compilation friendly
**Trade-off:** Slightly slower than CGO version, but simpler deployment
**Use Case:** Development, testing, lightweight deployments

---

### Visibility & Search

#### **Elasticsearch**
| Dependency | Version | Purpose | Assessment |
|------------|---------|---------|------------|
| `github.com/olivere/elastic/v7` | 7.0.32 | Elasticsearch client | ⚠️ Community client |

**Elasticsearch Version:** 7.10+
**Status:** Community-maintained (olivere), not official Elastic client
**Risk:** Medium - community support, API compatibility concerns
**Recommendation:** Monitor for Elasticsearch 8.x migration needs

---

### Observability

#### **Metrics**
| Dependency | Version | Purpose | Assessment |
|------------|---------|---------|------------|
| `github.com/uber-go/tally/v4` | 4.1.17 | Metrics (legacy) | ✅ Maintained by Uber |
| `github.com/prometheus/client_golang` | 1.21.0 | Prometheus exporter | ✅ Official |
| `go.opentelemetry.io/otel` | 1.34.0 | OpenTelemetry SDK | ✅ CNCF, active |
| `go.opentelemetry.io/otel/metric` | 1.34.0 | OTEL metrics | ✅ Modern standard |

**Strategy:** Dual support (Tally + OpenTelemetry)
**Migration Path:** Tally → OpenTelemetry
**Risk:** Low - both well-maintained

---

#### **Tracing**
| Dependency | Version | Purpose | Assessment |
|------------|---------|---------|------------|
| `go.opentelemetry.io/otel/trace` | 1.34.0 | Distributed tracing | ✅ CNCF standard |
| `go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc` | 0.59.0 | gRPC auto-instrumentation | ✅ Active |

**Risk:** Low - industry standard, CNCF project

---

#### **Logging**
| Dependency | Version | Purpose | Assessment |
|------------|---------|---------|------------|
| `go.uber.org/zap` | 1.27.0 | Structured logging | ✅ High-performance, Uber |

**Why Zap?**
- Zero-allocation logging
- Structured output (JSON)
- Excellent performance
- Production-proven

**Alternatives:** logrus, zerolog (but Zap is superior for high-throughput)

---

### Clustering & Membership

| Dependency | Version | Purpose | Assessment |
|------------|---------|---------|------------|
| `github.com/temporalio/ringpop-go` | 0.0.0-20250130... | Membership protocol | ✅ Temporal-maintained fork |
| `github.com/temporalio/tchannel-go` | 1.22.1-0.20240528... | RPC (Ringpop dependency) | ✅ Temporal-maintained fork |

**Origin:** Uber Ringpop (SWIM protocol-based gossip)
**Why Forked?** Temporal maintains fork for bug fixes and Go version compatibility
**Risk:** Low - owned by Temporal, production-proven

---

### Cloud Storage (Archival)

| Dependency | Version | Purpose | Assessment |
|------------|---------|---------|------------|
| `cloud.google.com/go/storage` | 1.51.0 | Google Cloud Storage | ✅ Official Google SDK |
| `github.com/aws/aws-sdk-go` | 1.55.8 | AWS S3 | ✅ Official AWS SDK (v1) |

**Note:** AWS SDK v1 (stable), not v2
**Recommendation:** Consider migrating to AWS SDK v2 for better performance

---

### Security & Authentication

| Dependency | Version | Purpose | Assessment |
|------------|---------|---------|------------|
| `github.com/golang-jwt/jwt/v4` | 4.5.2 | JWT tokens | ✅ Active fork of dgrijalva/jwt-go |
| `github.com/go-jose/go-jose/v4` | 4.0.5 | JOSE (JWE, JWS) | ✅ Active |
| `golang.org/x/crypto` | 0.37.0 | Cryptography | ✅ Official Go team |

**Risk:** Low - security-critical dependencies, all actively maintained

---

### Testing & Mocking

| Dependency | Version | Purpose | Assessment |
|------------|---------|---------|------------|
| `github.com/stretchr/testify` | 1.10.0 | Assertions, test suites | ✅ De facto standard |
| `go.uber.org/mock` | 0.6.0 | Mock generation | ✅ Uber (successor to gomock) |
| `github.com/golang/mock` | 1.6.0 | Mock generation (legacy) | ⚠️ Maintenance mode |

**Migration:** golang/mock → uber/mock (in progress)

---

### Utilities

| Dependency | Version | Purpose | Assessment |
|------------|---------|---------|------------|
| `github.com/google/uuid` | 1.6.0 | UUID generation | ✅ Google, standard |
| `github.com/robfig/cron/v3` | 3.0.1 | Cron expression parsing | ✅ Widely used |
| `github.com/sony/gobreaker` | 1.0.0 | Circuit breaker | ⚠️ Low activity, but stable |
| `golang.org/x/sync` | 0.16.0 | Concurrency utilities | ✅ Official Go team |
| `golang.org/x/time` | 0.10.0 | Rate limiting | ✅ Official Go team |

---

## Dependency Risk Assessment

### High-Risk Dependencies (Requires Attention)

**None identified** - all critical dependencies are actively maintained

### Medium-Risk Dependencies

| Dependency | Risk | Mitigation |
|------------|------|------------|
| `github.com/gocql/gocql` | Community-maintained | Monitor for issues, consider DataStax driver |
| `github.com/olivere/elastic/v7` | Community Elasticsearch client | Plan for Elasticsearch 8 migration |
| `github.com/lib/pq` | Maintenance mode | Complete migration to pgx/v5 |
| `github.com/sony/gobreaker` | Low activity | Small surface area, consider fork if needed |

### Low-Risk Dependencies

All other dependencies are officially maintained or have active community support.

---

## Internal Module Dependencies

### Service Dependency Graph

```
┌─────────────┐
│   cmd/      │  (Entry points)
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  temporal/  │  (Bootstrap & DI)
└──────┬──────┘
       │
       ▼
┌─────────────────────────────────────────┐
│            service/                     │
│  ┌─────────┬─────────┬─────────┬─────┐ │
│  │Frontend │ History │ Matching│Worker│ │
│  └─────────┴─────────┴─────────┴─────┘ │
└──────────────────┬──────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────┐
│            client/                       │
│  (Inter-service gRPC clients)            │
└──────────────────┬──────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────┐
│            common/                       │
│  ┌──────────────┬──────────────────┐    │
│  │ persistence  │ membership       │    │
│  │ metrics      │ dynamicconfig    │    │
│  │ log          │ namespace        │    │
│  │ rpc          │ authorization    │    │
│  └──────────────┴──────────────────┘    │
└──────────────────┬──────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────┐
│            api/                          │
│  (Generated protobuf code)               │
└──────────────────────────────────────────┘
```

### Dependency Rules

1. **Services MUST NOT import other services**
   - ✅ `service/frontend` → `client/history`
   - ❌ `service/frontend` → `service/history`

2. **Common packages MUST NOT import services**
   - ✅ `common/persistence` → `api/persistence`
   - ❌ `common/persistence` → `service/history`

3. **API layer has NO dependencies**
   - Generated code only imports standard library + protobuf

4. **Cmd layer can import anything**
   - Entry points wire up dependencies

---

## Common Package Dependencies

### High-Level Categories

```
common/
├── Foundation (no deps on other common packages)
│   ├── primitives/
│   ├── log/
│   ├── clock/
│   └── headers/
│
├── Infrastructure (depends on Foundation)
│   ├── persistence/
│   ├── membership/
│   ├── rpc/
│   ├── metrics/
│   └── dynamicconfig/
│
├── Business Logic (depends on Infrastructure)
│   ├── namespace/
│   ├── authorization/
│   ├── archiver/
│   ├── quotas/
│   └── searchattribute/
│
└── Service Utilities (depends on Business Logic)
    ├── resourcetest/
    └── testing/
```

### Critical Dependency Chains

#### Persistence Stack
```
service/history
    → common/persistence/client
        → common/persistence
            → common/persistence/cassandra
                → gocql/gocql
            → common/persistence/sql
                → jackc/pgx
                → go-sql-driver/mysql
```

#### Metrics Stack
```
service/frontend
    → common/metrics
        → uber-go/tally (if framework=tally)
        → go.opentelemetry.io/otel/metric (if framework=opentelemetry)
            → prometheus/client_golang (exporter)
```

#### RPC Stack
```
service/frontend
    → common/rpc
        → google.golang.org/grpc
            → go.opentelemetry.io/contrib/.../otelgrpc (tracing)
            → common/rpc/interceptor (rate limiting, retries)
```

---

## Dependency Management Best Practices

### Current Practices ✅

1. **Go Modules** - Modern dependency management
2. **Vendoring Disabled** - Rely on go.mod/go.sum
3. **Version Pinning** - Specific versions, not ranges
4. **Retracted Versions** - Explicitly retract bad releases (v1.26.0, v1.26.1)
5. **Indirect Dependencies** - Managed automatically by Go modules

### Recommendations 📋

1. **Dependency Auditing**
   ```bash
   go list -m all | go run golang.org/x/vuln/cmd/govulncheck@latest
   ```
   Regularly check for security vulnerabilities

2. **Dependency Updates**
   - Automated weekly checks via Dependabot/Renovate
   - Separate PRs for major, minor, patch updates
   - Run full test suite before merging

3. **Minimize Transitive Dependencies**
   - Review before adding heavy dependencies
   - Check `go mod graph` for transitive explosions

4. **Fork Strategy**
   - Fork critical dependencies (Ringpop, TChannel) for control
   - Maintain upstream sync process
   - Document fork reasons in go.mod comments

---

## Dependency License Compliance

### License Types

| License | Count | Examples |
|---------|-------|----------|
| **MIT** | ~60% | Temporal, testify, uuid |
| **Apache 2.0** | ~30% | gRPC, protobuf, OpenTelemetry |
| **BSD-3-Clause** | ~8% | Go standard library extensions |
| **BSD-2-Clause** | ~2% | pgx |

**Assessment:** ✅ All permissive licenses, no GPL/AGPL
**Commercial Use:** Unencumbered

---

## Security Vulnerability Tracking

### Known Issues (as of analysis date)

Check latest status:
```bash
go run golang.org/x/vuln/cmd/govulncheck@latest ./...
```

**Recommendation:** Setup automated CVE monitoring via:
- GitHub Dependabot alerts
- Snyk integration
- Weekly `govulncheck` runs in CI

---

## Dependency Update Strategy

### Cadence

| Dependency Type | Update Frequency | Strategy |
|----------------|------------------|----------|
| **Security patches** | Immediate | Auto-merge after CI |
| **Patch updates** | Weekly | Automated PR, manual review |
| **Minor updates** | Monthly | Review release notes, test |
| **Major updates** | Quarterly | Plan migration, extensive testing |

### Testing Requirements

Before merging dependency updates:
- ✅ Unit tests pass
- ✅ Integration tests pass (all databases)
- ✅ Functional tests pass
- ✅ Performance benchmarks show no regression
- ✅ Manual smoke testing

---

## Proposed Improvements

See RFCs for detailed proposals:

1. **[RFC-0008: Dependency Audit and Reduction](../rfcs/RFC-0008-dependency-audit.md)**
   - Reduce dependency count
   - Eliminate unmaintained dependencies
   - Standardize on single solutions (e.g., one PostgreSQL driver)

2. **[RFC-0009: Security Hardening](../rfcs/RFC-0009-security-hardening.md)**
   - Automated vulnerability scanning
   - Dependency update automation
   - Supply chain security (SLSA, Sigstore)

---

## Tools for Dependency Analysis

```bash
# List all dependencies
go list -m all

# Dependency graph
go mod graph

# Why is this dependency here?
go mod why github.com/some/dependency

# Find outdated dependencies
go list -u -m all

# Check for vulnerabilities
go run golang.org/x/vuln/cmd/govulncheck@latest ./...

# Visualize dependency graph
go install github.com/daveshanley/vacuum@latest
vacuum go-mod-graph --output graph.svg
```

---

**Next:** [Metrics Summary](metrics-summary.md) | [Back to Quick Start](00-quick-start.md)
