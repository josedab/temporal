# RFC-0003: Developer Experience Improvements

**Status:** Draft | **Priority:** P0 (Quick Win)
**Author:** Analysis Team | **Created:** 2025-11-16
**Effort:** 1 week | **Impact:** High

## Summary

Improve contributor onboarding and development experience through better documentation, tooling, and setup automation.

## Motivation

**Current Pain Points:**
- Setup takes 2-3 hours (confusing docs)
- Build errors not self-explanatory
- No IDE configuration guidance
- Contribution guide incomplete

**Impact:**
- Low external contribution rate
- New team members slow to ramp up
- Repeated questions in Slack

## Proposed Improvements

### 1. Automated Setup Script

```bash
#!/bin/bash
# scripts/dev-setup.sh

echo "Setting up Temporal development environment..."

# Check prerequisites
command -v go >/dev/null 2>&1 || { echo "Go required"; exit 1; }
command -v docker >/dev/null 2>&1 || { echo "Docker required"; exit 1; }

# Build
make bins

# Start dependencies
make start-dependencies

# Run tests
make unit-test

echo "✅ Setup complete! Run 'make start' to launch server."
```

**Time:** 5 minutes vs. 2-3 hours

### 2. IDE Configuration

**VSCode:** `.vscode/settings.json`
```json
{
  "go.testFlags": ["-v", "-race"],
  "go.buildTags": "disable_grpc_modules",
  "gopls": {
    "build.buildFlags": ["-tags=disable_grpc_modules"]
  }
}
```

**GoLand:** `.idea/runConfigurations/`
- Pre-configured run configurations
- Test configurations
- Debug configurations

### 3. Interactive Tutorial

```bash
temporal-dev tutorial

# Walks through:
# - Starting workflow
# - Implementing activity
# - Running tests
# - Debugging
```

### 4. Documentation Improvements

- Quick start guide (5 minutes to first workflow)
- Architecture diagrams (generated from code)
- Troubleshooting FAQ
- Video walkthroughs

## Implementation Plan

- **Day 1-2:** Setup script + IDE configs
- **Day 3-4:** Interactive tutorial
- **Day 5-6:** Documentation updates
- **Day 7:** Testing and feedback

## Success Metrics

- ✅ Setup time < 5 minutes
- ✅ Contributor satisfaction 4+/5
- ✅ 2x increase in contributions within 3 months

## Risks

None - purely additive improvements
