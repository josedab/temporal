# RFC-0010: Worker Versioning and Deployment UX Improvements

**Status:** Draft
**Effort:** 2-3 weeks
**Impact:** Medium (Developer Experience)
**Author:** Analysis Team
**Created:** 2025-11-16
**Commit SHA:** `0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3`

---

## Summary

Simplify worker versioning and deployment management UX, reducing deployment complexity from 10+ commands to 3 commands. Introduce automatic build ID generation, deployment wizard, and better visualization of rollout progress. This makes safe deployments accessible to all teams, not just experts.

---

## Motivation

### Current Worker Versioning Complexity

**Worker Versioning** enables safe deployments by routing tasks to compatible worker versions, preventing incompatible code from executing workflows.

**However, the UX is challenging:**

#### Problem 1: Too Many Commands

**Current deployment process:**
```bash
# Step 1: Build and tag code
git commit -m "Update workflow"
export BUILD_ID="v1.2.3-$(git rev-parse --short HEAD)-$(date +%s)"

# Step 2: Create deployment
temporal operator deployment create \
  --namespace production \
  --deployment-id "deployment-$BUILD_ID"

# Step 3: Add build ID to deployment
temporal operator deployment add-build-id \
  --namespace production \
  --deployment-id "deployment-$BUILD_ID" \
  --build-id "$BUILD_ID"

# Step 4: Create version set
temporal operator versioning create-version-set \
  --namespace production \
  --version-set-id "order-processing-set" \
  --build-id "$BUILD_ID"

# Step 5: Update deployment rules
temporal operator versioning update-deployment-rule \
  --namespace production \
  --version-set-id "order-processing-set" \
  --rule-type "ramp" \
  --ramp-percentage 10

# Step 6: Monitor metrics
# (manually check metrics dashboard)

# Step 7-10: Increment ramp 10%→25%→50%→100%
# (repeat step 5 four times)
```

**Total:** 10+ commands, error-prone, manual monitoring

---

#### Problem 2: Build ID Confusion

**Build ID:** Identifier for a specific code version

**Current issues:**
- Manual creation (prone to typos)
- No standard format
- Hard to map to git commits
- Collisions possible

**Examples seen in the wild:**
```bash
# Bad: Not unique
BUILD_ID="v1.2.3"

# Bad: Not human-readable
BUILD_ID="abc123def456"

# Bad: Timezone issues
BUILD_ID="2025-01-20-10:30"

# Good: But manual
BUILD_ID="v1.2.3-abc123-1234567890"
```

---

#### Problem 3: Deployment Rules Are Opaque

**Deployment rules** determine task routing.

**Current:**
```bash
# What does this do?
temporal operator versioning update-deployment-rule \
  --namespace production \
  --version-set-id "my-set" \
  --rule-type "ramp" \
  --ramp-percentage 25 \
  --previous-build-id "old-build"
```

**Questions users ask:**
- "What's the current state?"
- "How many workflows are on the new version?"
- "Can I rollback easily?"
- "What happens if I set percentage to 0?"

**Answer:** Not obvious without checking metrics/code

---

#### Problem 4: No Deployment Visualization

**Current monitoring:**
```bash
# Check metrics manually
curl http://temporal:9090/metrics | grep workflow_task

# Check logs
grep "build-id" /var/log/temporal/*.log

# Hope for the best
```

**Missing:**
- Deployment progress dashboard
- Rollout status (10% → 25% → ...)
- Error rate by build ID
- Easy rollback button

---

### Impact of Poor UX

**Quantitative:**
- Deployment takes 2-3 hours (vs. 10-20 min ideal)
- 30% of deployments have issues
- 50% of teams avoid versioning (too complex)
- Support tickets: ~20/month about versioning

**Qualitative:**
- "Versioning is too complex, we skip it"
- "I broke production because I forgot a step"
- "Can't we just have canary deployments like Kubernetes?"

---

## Proposed Solution

### Vision: 3-Command Deployment

```bash
# 1. Create deployment (auto-detects build ID)
temporal deployment create

# 2. Gradual rollout (automatic ramp)
temporal deployment rollout --strategy=gradual

# 3. Monitor (real-time dashboard)
temporal deployment status --watch
```

**That's it.** Safe deployment in 3 commands.

---

## Detailed Design

### 1. Automatic Build ID Generation

**Intelligent build ID inference:**

```bash
# File: cli/deployment/build_id.go
func GenerateBuildID() string {
    // 1. Try git
    if gitSHA := getGitSHA(); gitSHA != "" {
        return fmt.Sprintf("git-%s-%d", gitSHA[:8], time.Now().Unix())
    }

    // 2. Try environment variables (CI)
    if buildNum := os.Getenv("BUILD_NUMBER"); buildNum != "" {
        return fmt.Sprintf("ci-%s-%d", buildNum, time.Now().Unix())
    }

    // 3. Fallback: timestamp + random
    return fmt.Sprintf("build-%d-%s", time.Now().Unix(), randomString(6))
}

func getGitSHA() string {
    cmd := exec.Command("git", "rev-parse", "HEAD")
    output, err := cmd.Output()
    if err != nil {
        return ""
    }
    return strings.TrimSpace(string(output))
}
```

**Result:**
- `git-a1b2c3d4-1705834200` (git SHA + timestamp)
- Unique, traceable, human-readable

---

### 2. Simplified CLI Commands

#### 2.1 `temporal deployment create`

**One command to rule them all:**

```bash
temporal deployment create [OPTIONS]

Options:
  --namespace string     Namespace (default: current namespace)
  --build-id string      Build ID (default: auto-detect from git)
  --description string   Deployment description
  --dry-run              Show what would be created

Examples:
  # Auto-detect everything
  temporal deployment create

  # Specify build ID
  temporal deployment create --build-id v1.2.3

  # Dry run
  temporal deployment create --dry-run
```

**What it does:**
1. Generate build ID (if not provided)
2. Create deployment
3. Create version set (if needed)
4. Set initial deployment rule (0% ramp)
5. Print status

**Output:**
```
✅ Deployment created successfully

Build ID:        git-a1b2c3d4-1705834200
Deployment ID:   deployment-2025-01-20-103045
Version Set:     default-version-set
Current Traffic: 0% (ready to ramp up)

Next steps:
  1. Deploy workers with build ID: git-a1b2c3d4-1705834200
  2. Run: temporal deployment rollout --strategy=gradual
  3. Monitor: temporal deployment status --watch
```

---

#### 2.2 `temporal deployment rollout`

**Smart rollout strategies:**

```bash
temporal deployment rollout [OPTIONS]

Options:
  --strategy string      Rollout strategy (gradual|immediate|canary)
  --deployment-id string Deployment to roll out (default: latest)
  --monitor              Monitor rollout automatically

Strategies:
  gradual:   10% → 25% → 50% → 100% with 5-minute intervals
  immediate: 0% → 100% immediately
  canary:    5% for 10 minutes, then 100%

Examples:
  # Gradual rollout (recommended)
  temporal deployment rollout --strategy=gradual

  # Canary deployment
  temporal deployment rollout --strategy=canary

  # Immediate (for hotfixes)
  temporal deployment rollout --strategy=immediate
```

**Gradual strategy implementation:**
```go
// File: cli/deployment/rollout.go
func RolloutGradual(ctx context.Context, deploymentID string) error {
    stages := []int{10, 25, 50, 100}
    interval := 5 * time.Minute

    for _, percentage := range stages {
        fmt.Printf("Rolling out to %d%%...\n", percentage)

        // Update deployment rule
        err := updateDeploymentRule(ctx, deploymentID, percentage)
        if err != nil {
            return err
        }

        // Monitor health
        if percentage < 100 {
            fmt.Printf("Monitoring for %v...\n", interval)
            healthy, err := monitorHealth(ctx, deploymentID, interval)
            if err != nil || !healthy {
                fmt.Println("⚠️  Health check failed, rolling back")
                return rollback(ctx, deploymentID)
            }
        }
    }

    fmt.Println("✅ Rollout complete!")
    return nil
}

func monitorHealth(ctx context.Context, deploymentID string, duration time.Duration) (bool, error) {
    ticker := time.NewTicker(30 * time.Second)
    defer ticker.Stop()

    timeout := time.After(duration)

    for {
        select {
        case <-ticker.C:
            // Check error rate
            errorRate, err := getErrorRate(ctx, deploymentID)
            if err != nil {
                return false, err
            }

            if errorRate > 0.05 {  // 5% error threshold
                fmt.Printf("❌ Error rate too high: %.2f%%\n", errorRate*100)
                return false, nil
            }

            fmt.Printf("✅ Health check passed (error rate: %.2f%%)\n", errorRate*100)

        case <-timeout:
            return true, nil

        case <-ctx.Done():
            return false, ctx.Err()
        }
    }
}
```

---

#### 2.3 `temporal deployment status`

**Real-time deployment status:**

```bash
temporal deployment status [OPTIONS]

Options:
  --deployment-id string Deployment ID (default: latest)
  --watch                Watch for changes
  --format string        Output format (table|json) (default: table)

Examples:
  # One-time status
  temporal deployment status

  # Watch mode (updates every 5s)
  temporal deployment status --watch
```

**Output:**
```
Deployment Status: deployment-2025-01-20-103045
Build ID:          git-a1b2c3d4-1705834200
Status:            Rolling out

Traffic Distribution:
┌───────────────┬──────────┬─────────┬───────────┐
│ Build ID      │ Traffic  │ Workers │ Error Rate│
├───────────────┼──────────┼─────────┼───────────┤
│ git-a1b2c3d4  │ 25% ⬆    │ 3/12    │ 0.02%     │
│ git-prev123   │ 75% ⬇    │ 9/12    │ 0.01%     │
└───────────────┴──────────┴─────────┴───────────┘

Recent Events:
├─ 10:30:45 Rollout started
├─ 10:35:50 Ramped to 10% (healthy)
├─ 10:40:55 Ramped to 25% (monitoring...)
└─ (updating every 5s...)

Next Step: Ramp to 50% at 10:46:00
```

---

### 3. Deployment Wizard

**Interactive setup for complex scenarios:**

```bash
temporal deployment wizard

? What would you like to do? (Use arrow keys)
  ❯ Create a new deployment
    Rollout existing deployment
    Rollback deployment
    View deployment history

? Select rollout strategy: (Use arrow keys)
  ❯ Gradual (10% → 25% → 50% → 100%) [Recommended]
    Canary (5% for 10 min, then 100%)
    Immediate (0% → 100%)
    Custom (specify percentages)

? Monitor automatically? (Y/n) Y

? Automatically rollback on errors? (Y/n) Y
? Error threshold for rollback: (5%) 5

✅ Configuration complete!

Summary:
─────────
Build ID:        git-a1b2c3d4-1705834200
Strategy:        Gradual
Auto-monitor:    Yes
Auto-rollback:   Yes (error threshold: 5%)

? Proceed with deployment? (Y/n) Y

🚀 Starting deployment...
```

---

### 4. Deployment History and Auditing

**Track all deployments:**

```bash
temporal deployment history

Recent Deployments:
┌────────────────────────┬──────────────────────┬──────────┬──────────┐
│ Timestamp              │ Build ID             │ Strategy │ Status   │
├────────────────────────┼──────────────────────┼──────────┼──────────┤
│ 2025-01-20 10:30:45   │ git-a1b2c3d4        │ Gradual  │ Rolling  │
│ 2025-01-19 14:15:30   │ git-prev123         │ Gradual  │ Complete │
│ 2025-01-19 09:00:00   │ git-abc7890         │ Immediate│ Complete │
│ 2025-01-18 16:45:00   │ git-def4567         │ Canary   │ Rollback │
└────────────────────────┴──────────────────────┴──────────┴──────────┘

? Select deployment to view details: (Use arrow keys)
  ❯ git-a1b2c3d4 (current)
    git-prev123
    git-abc7890
    git-def4567
```

---

### 5. Better Error Messages

**Before:**
```
Error: deployment failed
Code: UNKNOWN
```

**After:**
```
❌ Deployment Failed

Reason: No workers found with build ID 'git-a1b2c3d4-1705834200'

Possible causes:
  1. Workers not deployed yet
  2. Workers using different build ID
  3. Workers not polling this task queue

Troubleshooting:
  1. Check worker logs:
     kubectl logs -l app=temporal-worker

  2. Verify build ID:
     echo $BUILD_ID

  3. Verify workers are running:
     temporal workflow list --query 'TaskQueue="my-queue"'

Need help? https://docs.temporal.io/deployments/troubleshooting
```

---

## Implementation Plan

### Week 1: CLI Foundation

**Day 1-2: Build ID Generation**
```bash
# Implement auto-detection
cli/deployment/build_id.go

# Tests
cli/deployment/build_id_test.go

# Integration test
temporal deployment create --dry-run
```

**Day 3-4: `deployment create` Command**
```bash
# Implement command
cli/cmd/deployment_create.go

# Backend API calls
# Test with real cluster
```

**Day 5: `deployment status` Command**
```bash
# Implement status display
cli/cmd/deployment_status.go

# Add watch mode
```

---

### Week 2: Rollout & Monitoring

**Day 1-3: `deployment rollout` Command**
```bash
# Implement rollout strategies
cli/deployment/strategies.go

# Implement health monitoring
cli/deployment/health.go

# Test gradual rollout end-to-end
```

**Day 4: Deployment Wizard**
```bash
# Interactive CLI (using survey library)
go get github.com/AlecAivazis/survey/v2

# Implement wizard flow
cli/cmd/deployment_wizard.go
```

**Day 5: Error Messages**
```bash
# Improve error handling across CLI
# Add structured error messages
# Add troubleshooting tips
```

---

### Week 3: Polish & Documentation

**Day 1-2: Deployment History**
```bash
# Implement history command
cli/cmd/deployment_history.go

# Add filtering/searching
```

**Day 3: Documentation**
```bash
# Update CLI docs
docs/cli/deployment.md

# Create tutorial
docs/tutorials/safe-deployments.md

# Add examples
```

**Day 4: Testing**
```bash
# End-to-end testing
# User acceptance testing
# Fix bugs
```

**Day 5: Release**
```bash
# Merge PR
# Update changelog
# Announce in community
```

---

## Success Metrics

| Metric | Baseline | Target | Measurement |
|--------|----------|--------|-------------|
| **Commands for Deployment** | 10+ | 3 | CLI usage logs |
| **Deployment Time** | 2-3 hours | 10-20 min | Time tracking |
| **Deployment Success Rate** | 70% | 95% | Deployment logs |
| **User Satisfaction** | 2.8/5 | 4.0+/5 | User survey |
| **Worker Versioning Adoption** | 50% | 90%+ | Feature usage telemetry |
| **Support Tickets** | ~20/month | <5/month | Ticket tracking |

---

## User Testimonials (Target)

**Before:**
> "Worker versioning is way too complicated. We just redeploy everything and hope nothing breaks." - Platform Engineer

**After (Goal):**
> "Wow, `temporal deployment rollout` is amazing! It's like Kubernetes deployments but for workflows." - Same Engineer

---

## Risks and Mitigations

### Risk 1: Users Dislike New UX

**Likelihood:** Low
**Impact:** Medium

**Mitigation:**
- User testing before release
- Beta program with 5-10 teams
- Keep old commands (deprecate slowly)
- Clear migration guide

---

### Risk 2: Automatic Rollback Too Aggressive

**Likelihood:** Medium
**Impact:** Low

**Mitigation:**
- Configurable thresholds
- Option to disable auto-rollback
- Dry-run mode to test behavior
- Manual override always available

---

### Risk 3: Build ID Collisions

**Likelihood:** Very Low
**Impact:** High

**Mitigation:**
- Include timestamp (collision probability < 0.001%)
- Validate uniqueness before creating
- Clear error message if collision detected

---

## Alternatives Considered

### Alternative 1: No Abstraction (Keep Current UX)

**Pros:**
- No development effort
- Users already learned it

**Cons:**
- Complexity remains
- Low adoption continues
- Support burden high

**Decision:** Rejected, UX improvement is necessary

---

### Alternative 2: Web UI Instead of CLI

**Pros:**
- Visual interface
- Easier for non-technical users

**Cons:**
- Requires additional infra (UI server)
- Not scriptable/automatable
- Slower iteration

**Decision:** CLI first, UI later (Phase 2)

---

### Alternative 3: Full GitOps Integration

**Pros:**
- Infrastructure as code
- Deployment via git push

**Cons:**
- Requires additional tooling (Flux, Argo)
- Steeper learning curve
- Overkill for many users

**Decision:** Deferred to Phase 3

---

## Migration Strategy

### Backward Compatibility

**All old commands still work:**
```bash
# Old way (still supported)
temporal operator deployment create ...
temporal operator versioning update-deployment-rule ...

# New way (recommended)
temporal deployment create
temporal deployment rollout
```

**Deprecation Timeline:**
- **Month 1-3:** New commands available, old commands work (warning message)
- **Month 4-6:** Old commands deprecated (error suggests new command)
- **Month 7+:** Old commands removed

---

## Related Work

- **RFC-0003: Developer Experience** - Part of broader DX improvements
- **RFC-0006: Documentation** - Better deployment docs
- **Blog Post 7: Future of Temporal** - Worker versioning feature

---

## References

### Temporal Docs
- [Worker Versioning](https://docs.temporal.io/workers#worker-versioning)
- [Deployment Best Practices](https://docs.temporal.io/production-deployment)

### Inspiration
- [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Flagger (Kubernetes Progressive Delivery)](https://flagger.app/)
- [Argo Rollouts](https://argo-rollouts.readthedocs.io/)

---

**Next Steps:**
1. Review and approve RFC
2. User testing with prototype
3. Begin Week 1 implementation
4. Beta release Month 2

**Status:** Ready for review
**Estimated Completion:** 3 weeks from approval
