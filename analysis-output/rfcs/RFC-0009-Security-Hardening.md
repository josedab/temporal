# RFC-0009: Security Hardening and Compliance

**Status:** Draft
**Effort:** 8-12 weeks
**Impact:** High (Critical for Enterprise)
**Author:** Analysis Team
**Created:** 2025-11-16
**Commit SHA:** `0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3`

---

## Summary

Implement comprehensive security hardening measures including automated vulnerability scanning, supply chain security (SLSA Level 3), security audit tooling, incident response procedures, and compliance frameworks (SOC2/ISO 27001 alignment). This RFC establishes Temporal as a security-first platform suitable for enterprise deployments handling sensitive data.

---

## Motivation

### Current Security Posture

**Strengths ✅:**
- TLS support for all communication
- Authentication/authorization framework exists
- No hardcoded secrets (verified)
- Code review required for all changes
- Dependency vulnerability scanning (manual)

**Gaps ❌:**

#### 1. No Automated Security Scanning

**Current State:**
```bash
# Manual vulnerability check
go run golang.org/x/vuln/cmd/govulncheck@latest ./...

# Run occasionally, not in CI
```

**Issues:**
- Vulnerabilities discovered days/weeks after disclosure
- No SAST (Static Application Security Testing)
- No DAST (Dynamic Application Security Testing)
- No container image scanning

**Target:**
- Automated scanning in CI/CD
- Real-time alerts for CVEs
- Shift-left security (catch issues before merge)

---

#### 2. No Supply Chain Security

**Risk:** Compromised dependencies or build process

**Missing:**
- **SLSA Compliance:** Supply Chain Levels for Software Artifacts
- **Binary Signing:** No cryptographic verification of releases
- **SBOM:** No Software Bill of Materials
- **Reproducible Builds:** Builds not deterministic

**Example Attack Vector:**
```
1. Attacker compromises build server
2. Injects malicious code during build
3. Releases signed by compromised key
4. Users download and run compromised binary
```

**Target:** SLSA Level 3 compliance

---

#### 3. No Penetration Testing

**Current:** No regular security audits

**Concern:**
- Unknown vulnerabilities in production
- Compliance requirements not met (SOC2, ISO 27001)
- Customer security questionnaires difficult to answer

**Target:** Annual third-party penetration test

---

#### 4. Incident Response Procedures Missing

**If security incident occurs:**
- No documented response plan
- No designated security team
- No vulnerability disclosure policy
- No post-mortem template

**Impact:**
- Slow response to incidents
- Inconsistent communication
- Potential legal/PR issues

---

#### 5. Compliance Framework Gaps

**Enterprise Requirements:**
- **SOC2 Type II:** Most common requirement
- **ISO 27001:** International standard
- **GDPR:** For EU customers
- **HIPAA:** For healthcare
- **PCI-DSS:** For payment processing

**Current:** No formal compliance program

---

## Proposed Solution

### Security Architecture

```
┌─────────────────────────────────────────────────────┐
│         Security Hardening Architecture             │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Layer 1: Automated Scanning                        │
│  ├─ govulncheck (CVE scanning)                      │
│  ├─ Snyk (SAST + dependency scanning)              │
│  ├─ Trivy (container scanning)                      │
│  ├─ gosec (Go-specific security checks)            │
│  └─ CodeQL (GitHub advanced security)              │
│                                                     │
│  Layer 2: Supply Chain Security                     │
│  ├─ SLSA Level 3 compliance                         │
│  ├─ Sigstore signing (cosign)                       │
│  ├─ SBOM generation (syft)                          │
│  └─ Reproducible builds                             │
│                                                     │
│  Layer 3: Runtime Security                          │
│  ├─ mTLS enforcement                                │
│  ├─ RBAC enhancements                               │
│  ├─ Audit logging                                   │
│  └─ Rate limiting & DDoS protection                 │
│                                                     │
│  Layer 4: Governance                                │
│  ├─ Security runbooks                               │
│  ├─ Incident response plan                          │
│  ├─ Vulnerability disclosure policy                 │
│  └─ Compliance documentation                        │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## Detailed Design

### 1. Automated Security Scanning

#### 1.1 Multi-Layer Scanning Approach

```yaml
# .github/workflows/security-comprehensive.yml
name: Comprehensive Security Scan

on:
  push:
    branches: [main]
  pull_request:
  schedule:
    - cron: '0 0 * * *'  # Daily

jobs:
  vulnerability-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: govulncheck (Go vulnerabilities)
        run: |
          go install golang.org/x/vuln/cmd/govulncheck@latest
          govulncheck -test ./...

      - name: Snyk Security Scan
        uses: snyk/actions/golang@master
        with:
          command: test
          args: --severity-threshold=high
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}

      - name: gosec (Go Security Checker)
        uses: securego/gosec@master
        with:
          args: '-exclude-generated ./...

'

      - name: CodeQL Analysis
        uses: github/codeql-action/analyze@v2
        with:
          language: go

  container-scan:
    runs-on: ubuntu-latest
    steps:
      - name: Build container
        run: docker build -t temporal-server:test .

      - name: Trivy container scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'temporal-server:test'
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'

      - name: Upload Trivy results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'

  secrets-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Full history for scanning

      - name: TruffleHog Secret Scanning
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          head: HEAD
```

---

#### 1.2 Security Dashboard

**Tool:** Security Command Center (internal dashboard)

```go
// tools/security-dashboard/main.go
package main

// Aggregates security scan results from:
// - govulncheck
// - Snyk
// - Trivy
// - gosec
// - CodeQL

// Displays:
// - CVE count by severity
// - Trend over time
// - MTTR (Mean Time To Remediate)
// - Open security issues
```

**Example Dashboard:**
```
Temporal Security Dashboard
===========================

CVE Status:
├─ Critical: 0 ✅
├─ High: 0 ✅
├─ Medium: 2 ⚠️ (remediation in progress)
└─ Low: 5 (accepted risk)

Supply Chain:
├─ SLSA Level: 3 ✅
├─ SBOM: Generated ✅
└─ Signatures: Valid ✅

Compliance:
├─ SOC2: In progress (85% complete)
├─ ISO 27001: Not started
└─ Penetration Test: Last run 3 months ago
```

---

### 2. Supply Chain Security (SLSA Level 3)

#### 2.1 SLSA Requirements

**SLSA Levels:**
- **Level 1:** Build process documented
- **Level 2:** Signed provenance
- **Level 3:** Hardened build platform, non-falsifiable provenance
- **Level 4:** Two-person review, hermetic builds

**Target:** Level 3

---

#### 2.2 Provenance Generation

**Using SLSA GitHub Generator:**

```yaml
# .github/workflows/release.yml
name: Release with SLSA Provenance

on:
  push:
    tags:
      - 'v*'

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      digest: ${{ steps.build.outputs.digest }}
    steps:
      - uses: actions/checkout@v4

      - name: Build binary
        id: build
        run: |
          make bins
          sha256sum temporal-server > digest.txt
          echo "digest=$(cat digest.txt)" >> $GITHUB_OUTPUT

      - name: Upload binary
        uses: actions/upload-artifact@v3
        with:
          name: temporal-server
          path: temporal-server

  provenance:
    needs: [build]
    permissions:
      actions: read
      id-token: write
      contents: write
    uses: slsa-framework/slsa-github-generator/.github/workflows/generator_generic_slsa3.yml@v1.9.0
    with:
      base64-subjects: "${{ needs.build.outputs.digest }}"
      upload-assets: true
```

**Result:** `temporal-server-v1.26.0.intoto.jsonl` (provenance file)

---

#### 2.3 Binary Signing with Sigstore

**Sign releases cryptographically:**

```yaml
jobs:
  sign:
    runs-on: ubuntu-latest
    steps:
      - name: Install cosign
        uses: sigstore/cosign-installer@v3

      - name: Sign binary
        run: |
          cosign sign-blob \
            --yes \
            --bundle temporal-server.bundle \
            temporal-server

      - name: Verify signature
        run: |
          cosign verify-blob \
            --bundle temporal-server.bundle \
            temporal-server
```

**Users can verify:**
```bash
# Download binary and signature
curl -LO https://github.com/temporalio/temporal/releases/download/v1.26.0/temporal-server
curl -LO https://github.com/temporalio/temporal/releases/download/v1.26.0/temporal-server.bundle

# Verify
cosign verify-blob \
  --bundle temporal-server.bundle \
  --certificate-identity-regexp="https://github.com/temporalio/temporal/*" \
  --certificate-oidc-issuer=https://token.actions.githubusercontent.com \
  temporal-server
```

---

#### 2.4 SBOM Generation

**Generate Software Bill of Materials:**

```yaml
jobs:
  sbom:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Generate SBOM with syft
        uses: anchore/sbom-action@v0
        with:
          path: ./
          format: spdx-json
          output-file: temporal-sbom.spdx.json

      - name: Upload SBOM
        uses: actions/upload-artifact@v3
        with:
          name: sbom
          path: temporal-sbom.spdx.json

      - name: Attach to release
        uses: softprops/action-gh-release@v1
        with:
          files: temporal-sbom.spdx.json
```

**SBOM Format (SPDX):**
```json
{
  "spdxVersion": "SPDX-2.3",
  "name": "temporal-server",
  "packages": [
    {
      "name": "github.com/temporalio/temporal",
      "versionInfo": "v1.26.0",
      "licenseConcluded": "MIT"
    },
    {
      "name": "google.golang.org/grpc",
      "versionInfo": "v1.72.2",
      "licenseConcluded": "Apache-2.0"
    }
    // ... all dependencies
  ]
}
```

---

### 3. Runtime Security

#### 3.1 mTLS Enforcement

**Current:** TLS optional
**Target:** mTLS mandatory for production

```yaml
# config/production.yaml
global:
  tls:
    # Server TLS
    server:
      certFile: /etc/temporal/certs/server-cert.pem
      keyFile: /etc/temporal/certs/server-key.pem
      clientCAFiles:
        - /etc/temporal/certs/ca-cert.pem
      requireClientAuth: true  # ✅ Enforce mTLS

    # Inter-service TLS
    internode:
      certFile: /etc/temporal/certs/internode-cert.pem
      keyFile: /etc/temporal/certs/internode-key.pem
      clientCAFiles:
        - /etc/temporal/certs/ca-cert.pem
      requireClientAuth: true  # ✅ Enforce mTLS
```

---

#### 3.2 Enhanced RBAC

**Current:** Basic authorization
**Proposed:** Fine-grained permissions

```go
// File: common/authorization/permissions.go
const (
    // Workflow Permissions
    PermissionStartWorkflow       = "temporal.workflow.start"
    PermissionTerminateWorkflow   = "temporal.workflow.terminate"
    PermissionSignalWorkflow      = "temporal.workflow.signal"
    PermissionQueryWorkflow       = "temporal.workflow.query"

    // Admin Permissions
    PermissionCreateNamespace     = "temporal.namespace.create"
    PermissionDeleteNamespace     = "temporal.namespace.delete"
    PermissionListWorkflows       = "temporal.workflow.list"

    // Operator Permissions
    PermissionViewMetrics         = "temporal.metrics.view"
    PermissionModifyConfig        = "temporal.config.modify"
)

// Role definitions
var (
    RoleUser = Role{
        Permissions: []string{
            PermissionStartWorkflow,
            PermissionSignalWorkflow,
            PermissionQueryWorkflow,
        },
    }

    RoleAdmin = Role{
        Permissions: []string{
            // All user permissions +
            PermissionTerminateWorkflow,
            PermissionCreateNamespace,
            PermissionListWorkflows,
        },
    }

    RoleOperator = Role{
        Permissions: []string{
            // All admin permissions +
            PermissionViewMetrics,
            PermissionModifyConfig,
        },
    }
)
```

---

#### 3.3 Comprehensive Audit Logging

**Log all security-relevant events:**

```go
// File: common/audit/logger.go
package audit

type AuditEvent struct {
    Timestamp  time.Time
    User       string
    Action     string
    Resource   string
    Result     string  // success/failure
    IPAddress  string
    Metadata   map[string]string
}

// Examples of logged events:
// - User authentication (success/failure)
// - Workflow start/terminate
// - Namespace create/delete
// - Configuration changes
// - Permission denied events

func LogAuditEvent(ctx context.Context, event AuditEvent) {
    // Write to dedicated audit log
    // Format: JSON for parsing
    // Retention: 7 years (compliance requirement)
    // Immutable: append-only, cannot be modified
}
```

**Audit Log Format:**
```json
{
  "timestamp": "2025-01-20T10:30:00Z",
  "user": "alice@example.com",
  "action": "temporal.workflow.terminate",
  "resource": "workflow/order-12345",
  "result": "success",
  "ip_address": "192.168.1.100",
  "metadata": {
    "namespace": "production",
    "reason": "manual intervention"
  }
}
```

---

### 4. Security Governance

#### 4.1 Incident Response Plan

**File:** `docs/security/INCIDENT_RESPONSE.md`

```markdown
# Security Incident Response Plan

## Severity Levels

**Critical (P0):**
- Active exploit in the wild
- Data breach
- Complete service outage due to security issue

**High (P1):**
- Vulnerability with high CVSS (7.0-10.0)
- Potential data exposure

**Medium (P2):**
- Vulnerability with medium CVSS (4.0-6.9)

**Low (P3):**
- Vulnerability with low CVSS (<4.0)

## Response Timeline

| Severity | Initial Response | Fix Released | Communication |
|----------|-----------------|--------------|---------------|
| P0 | 1 hour | 24 hours | Immediate |
| P1 | 4 hours | 7 days | Within 24h |
| P2 | 1 business day | 30 days | After fix |
| P3 | 1 week | 90 days | After fix |

## Response Steps

### 1. Detection
- Automated scan finds vulnerability
- User reports security issue
- Third-party researcher disclosure

### 2. Triage (< 1 hour for P0)
- Assess severity
- Identify affected versions
- Determine impact

### 3. Containment
- If actively exploited: disable feature via dynamic config
- Notify security-team channel
- Brief leadership

### 4. Remediation
- Develop fix
- Test thoroughly
- Prepare security advisory

### 5. Release
- Cut security release
- Update security advisory
- Notify affected users

### 6. Post-Mortem
- Root cause analysis
- Process improvements
- Update runbooks
```

---

#### 4.2 Vulnerability Disclosure Policy

**File:** `SECURITY.md` (GitHub recognizes this)

```markdown
# Security Policy

## Supported Versions

| Version | Supported |
|---------|-----------|
| 1.26.x  | ✅ Yes    |
| 1.25.x  | ✅ Yes    |
| 1.24.x  | ⚠️ Security patches only |
| < 1.24  | ❌ No     |

## Reporting a Vulnerability

**DO NOT** open a public GitHub issue for security vulnerabilities.

### How to Report

1. Email: security@temporal.io
2. PGP Key: [Download](https://temporal.io/security.asc)
3. Bug Bounty: [HackerOne Program](https://hackerone.com/temporal)

### What to Include

- Description of vulnerability
- Steps to reproduce
- Affected versions
- Proposed fix (if any)

### What to Expect

- **Acknowledgment:** Within 24 hours
- **Triage:** Within 72 hours
- **Fix Timeline:** Based on severity (see above)
- **Credit:** Acknowledged in security advisory (if desired)

## Bug Bounty Program

We run a private bug bounty program on HackerOne.

**Rewards:**
- Critical: $5,000 - $15,000
- High: $1,000 - $5,000
- Medium: $500 - $1,000
- Low: $100 - $500

**Scope:**
- temporalio/temporal (server)
- temporalio/ui (web UI)
- temporalio/cli (command line)

**Out of Scope:**
- Denial of Service
- Social engineering
- Physical attacks
```

---

### 5. Compliance Framework

#### 5.1 SOC2 Type II Preparation

**Requirements for SOC2:**

**Trust Service Criteria:**
1. **Security:** Protect against unauthorized access
2. **Availability:** System available for operation
3. **Processing Integrity:** System achieves objectives
4. **Confidentiality:** Confidential info protected
5. **Privacy:** Personal info protected

**Implementation Checklist:**

```markdown
## Security
- [x] Access controls (RBAC)
- [x] Encryption in transit (TLS)
- [x] Encryption at rest (database)
- [x] Audit logging
- [ ] Annual penetration test
- [ ] Vulnerability management program
- [ ] Incident response plan (documented)

## Availability
- [x] High availability architecture
- [x] Disaster recovery plan
- [x] Monitoring and alerting
- [ ] SLA documentation
- [ ] Change management process

## Processing Integrity
- [x] Code review requirements
- [x] CI/CD with automated tests
- [ ] Data validation controls
- [ ] Error handling and logging

## Confidentiality
- [x] Namespace isolation
- [x] TLS for all communication
- [ ] Data classification policy
- [ ] Retention and disposal procedures

## Privacy (if applicable)
- [ ] GDPR compliance (for EU customers)
- [ ] Data processing agreements
- [ ] Privacy policy
```

---

#### 5.2 ISO 27001 Alignment

**Key Controls:**

```markdown
## A.9: Access Control
- ✅ User authentication (OAuth, OIDC)
- ✅ Role-based access control
- ⚠️ Multi-factor authentication (MFA) - roadmap

## A.10: Cryptography
- ✅ TLS 1.3 for data in transit
- ✅ Database encryption at rest
- ✅ Key management (KMS integration)

## A.12: Operations Security
- ✅ Change management (CI/CD)
- ✅ Capacity management (monitoring)
- ⚠️ Backup verification - improve

## A.14: System Acquisition
- ✅ Secure development lifecycle
- ✅ Code review mandatory
- ✅ Security testing in CI
```

---

## Implementation Plan

### Phase 1: Automated Scanning (Weeks 1-2)

**Week 1: CI Integration**
```bash
# Add security workflows
cp security/templates/*.yml .github/workflows/

# Configure Snyk
snyk auth $SNYK_TOKEN
snyk test

# Configure CodeQL
# GitHub Settings → Security → Code scanning → Setup CodeQL
```

**Week 2: Dashboard & Alerting**
```bash
# Setup security dashboard
cd tools/security-dashboard
go build
./security-dashboard --port 8080

# Configure Slack alerts
# Add webhook to .github/workflows/
```

---

### Phase 2: Supply Chain Security (Weeks 3-5)

**Week 3: SLSA Setup**
```bash
# Add SLSA provenance generation
# Update release workflow

# Test locally
act -W .github/workflows/release.yml
```

**Week 4: Sigstore Integration**
```bash
# Install cosign
go install github.com/sigstore/cosign/v2/cmd/cosign@latest

# Generate test signature
cosign sign-blob temporal-server

# Update docs
docs/security/VERIFICATION.md
```

**Week 5: SBOM Generation**
```bash
# Install syft
curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh

# Generate SBOM
syft packages . -o spdx-json > sbom.json

# Integrate into release workflow
```

---

### Phase 3: Security Runbooks (Weeks 6-9)

**Week 6-7: Documentation**
- Create SECURITY.md
- Document incident response plan
- Write security runbooks

**Week 8: Training**
- Team training on incident response
- Security champion program
- Tabletop exercise

**Week 9: Process Setup**
- Setup security-team channel
- Create on-call rotation
- Establish metrics dashboard

---

### Phase 4: Penetration Testing (Weeks 10-12)

**Week 10: Preparation**
- Select vendor (e.g., NCC Group, Trail of Bits)
- Define scope
- Prepare environment

**Week 11-12: Testing**
- Vendor conducts test
- Fix critical findings
- Remediation validation

---

## Success Metrics

| Metric | Baseline | Target | Measurement |
|--------|----------|--------|-------------|
| **CVE Detection Time** | 7 days | 1 day | Automated scanning |
| **CVE Remediation Time (Critical)** | 14 days | 3 days | GitHub issues |
| **SLSA Level** | 0 | 3 | SLSA assessment |
| **Security Scan Coverage** | 0% | 100% | CI coverage |
| **Penetration Test Results** | N/A | 0 critical findings | Vendor report |
| **SOC2 Readiness** | 0% | 85%+ | Audit checklist |

---

## Risks and Mitigations

### Risk 1: Performance Impact

**Concern:** Security checks slow down CI

**Mitigation:**
- Run security scans in parallel
- Cache scan results
- Optimize scanner configurations
- Run full scans nightly, quick scans on PR

---

### Risk 2: False Positives

**Concern:** Alerts overwhelm team

**Mitigation:**
- Tune scanners (suppress known false positives)
- Triage process (security champion reviews)
- Allowlist for accepted risks
- Regular review of suppressed alerts

---

### Risk 3: Compliance Cost

**Concern:** SOC2 audit expensive ($50-100K)

**Mitigation:**
- Phase compliance work over time
- Self-assess first (readiness assessment)
- Bundle with other audits if possible
- Consider cloud provider compliance inheritance

---

## Alternatives Considered

### Alternative 1: Buy Security Platform

**Option:** Use platform like Snyk, GitLab Ultimate, or GitHub Advanced Security

**Pros:**
- Comprehensive tooling
- Managed service

**Cons:**
- Expensive ($50-100K/year)
- Vendor lock-in

**Decision:** Hybrid approach (use some commercial tools, supplement with open source)

---

### Alternative 2: Minimal Security

**Option:** Only address CVEs as discovered

**Pros:**
- Low effort
- No tooling costs

**Cons:**
- **High risk** (reactive, not proactive)
- Won't pass enterprise security reviews
- No compliance certification possible

**Decision:** Rejected, security is critical for enterprise adoption

---

## Related Work

- **RFC-0008: Dependency Audit** - Complements supply chain security
- **RFC-0006: Documentation** - Security runbooks included
- **Blog Post 6: Production Operations** - Security best practices

---

## References

### Standards
- [SLSA Framework](https://slsa.dev/)
- [SOC2 Trust Service Criteria](https://www.aicpa.org/soc)
- [ISO 27001](https://www.iso.org/isoiec-27001-information-security.html)

### Tools
- [Snyk](https://snyk.io/)
- [Trivy](https://github.com/aquasecurity/trivy)
- [Sigstore](https://www.sigstore.dev/)
- [Syft](https://github.com/anchore/syft)

---

**Next Steps:**
1. Review and approve RFC
2. Allocate security engineering resources
3. Begin Phase 1 (automated scanning)
4. Establish security-team channel

**Status:** Ready for review
**Estimated Completion:** 12 weeks from approval
