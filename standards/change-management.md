# STD-011: Change Management Standard

**Standard ID:** STD-011
**Version:** 1.0
**Status:** Active
**Effective Date:** 2026-02-06

---

## 1. Purpose

This standard establishes the processes, procedures, and best practices for managing changes to Salesforce environments. It ensures controlled, documented, and reversible deployments that minimize risk to production systems.

## 2. Scope

This standard applies to:
- Metadata deployments (Apex, LWC, Flows, Objects, etc.)
- Configuration changes
- Data migrations and updates
- Package installations and upgrades
- Permission and security changes
- Integration changes

## 3. Environment Strategy

### 3.1 Standard Environment Model

```
┌─────────────────────────────────────────────────────────────────┐
│                         PRODUCTION                               │
│                    (Live business data)                         │
└─────────────────────────────────────────────────────────────────┘
                              ▲
                              │ Deploy (Change Set / CLI / Pipeline)
                              │
┌─────────────────────────────────────────────────────────────────┐
│                         STAGING/UAT                              │
│               (Production-like for final testing)               │
└─────────────────────────────────────────────────────────────────┘
                              ▲
                              │ Promote
                              │
┌─────────────────────────────────────────────────────────────────┐
│                        QA/TESTING                                │
│                  (Integration testing)                          │
└─────────────────────────────────────────────────────────────────┘
                              ▲
                              │ Promote
                              │
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│  Developer    │  │  Developer    │  │  Developer    │
│  Sandbox 1    │  │  Sandbox 2    │  │  Sandbox 3    │
└───────────────┘  └───────────────┘  └───────────────┘
```

### 3.2 Sandbox Types and Usage

| Sandbox Type | Use Case | Refresh Frequency | Data |
|--------------|----------|-------------------|------|
| Developer | Individual development | As needed | No data |
| Developer Pro | Complex development, testing | Weekly | Limited data |
| Partial Copy | Integration testing, QA | Monthly | Sampled data |
| Full Copy | UAT, Staging, Performance | Quarterly | Full data copy |

### 3.3 Sandbox Naming Convention

```
Format: [OrgName]-[Type]-[Purpose/Team]

Examples:
✓ acme-dev-johndoe
✓ acme-dev-team-sales
✓ acme-qa-integration
✓ acme-uat-release-2026q1
✓ acme-full-staging
```

## 4. Change Classification

### 4.1 Change Types

| Type | Risk Level | Approval Required | Examples |
|------|------------|-------------------|----------|
| Standard | Low | None/Auto | Minor bug fixes, label changes |
| Normal | Medium | Team Lead | New features, moderate changes |
| Emergency | Variable | Fast-track | Critical production fixes |
| Major | High | CAB | Architecture changes, integrations |

### 4.2 Risk Assessment Matrix

| Factor | Low | Medium | High |
|--------|-----|--------|------|
| Data Impact | No data changes | Updates existing data | Creates/deletes data |
| User Impact | No user-facing changes | Minor UI changes | Major workflow changes |
| Integration | No integration changes | Existing integration updates | New integrations |
| Rollback | Easy rollback | Requires manual steps | Difficult/impossible |
| Test Coverage | >90% coverage | 75-90% coverage | <75% coverage |

> **Note:** Test coverage percentages above are for **risk assessment** purposes. For actual coverage requirements, see [STD-009: Testing Standards](./testing-standards.md) Section 3.1.

### 4.3 Change Request Template

```markdown
## Change Request

**Title:** [Brief description]
**Requestor:** [Name]
**Date:** [YYYY-MM-DD]
**Change Type:** [Standard/Normal/Emergency/Major]
**Target Environment:** [Production/UAT/QA]
**Requested Deploy Date:** [YYYY-MM-DD HH:MM]

### Description
[Detailed description of the change]

### Business Justification
[Why is this change needed?]

### Technical Details
- Components: [List of metadata components]
- Dependencies: [Other changes/packages required]
- Data Impact: [None/Read/Write/Delete]

### Risk Assessment
- Risk Level: [Low/Medium/High]
- Rollback Plan: [How to reverse if needed]
- Affected Users: [User groups impacted]

### Testing
- Test Plan: [Link to test cases]
- UAT Sign-off: [Name and date]

### Deployment Steps
1. [Step 1]
2. [Step 2]
3. [Post-deployment verification]

### Approvals
- [ ] Technical Review: [Name]
- [ ] Business Owner: [Name]
- [ ] Release Manager: [Name]
```

## 5. Development Workflow

### 5.1 Git Branching Strategy

```
main (production)
│
├── release/2026-Q1 (release branch)
│   │
│   ├── feature/JIRA-123-new-feature
│   ├── feature/JIRA-456-another-feature
│   └── bugfix/JIRA-789-fix-issue
│
└── hotfix/JIRA-999-critical-fix (emergency)
```

### 5.2 Branch Naming Convention

```
Format: [type]/[ticket-id]-[description]

Types:
- feature/ - New functionality
- bugfix/ - Bug fixes
- hotfix/ - Emergency production fixes
- refactor/ - Code refactoring
- docs/ - Documentation updates

Examples:
✓ feature/JIRA-123-account-automation
✓ bugfix/JIRA-456-contact-validation
✓ hotfix/JIRA-789-critical-security-fix
```

### 5.3 Commit Message Format

```
Format: [TICKET-ID] [Type]: [Description]

Types:
- feat: New feature
- fix: Bug fix
- refactor: Code refactoring
- docs: Documentation
- test: Testing
- chore: Maintenance

Examples:
✓ JIRA-123 feat: Add account tier calculation flow
✓ JIRA-456 fix: Resolve null pointer in contact trigger
✓ JIRA-789 refactor: Optimize batch query performance
```

### 5.4 Pull Request Process

```markdown
## Pull Request Template

**Ticket:** [JIRA-XXX]
**Type:** [Feature/Bugfix/Hotfix]

### Description
[What does this PR do?]

### Changes
- [Component 1]: [Description of change]
- [Component 2]: [Description of change]

### Testing
- [ ] Unit tests pass locally
- [ ] Apex test coverage meets requirements
- [ ] Tested in developer sandbox
- [ ] LWC/Aura component tested

### Deployment Notes
- [ ] No special deployment steps required
- [ ] Requires post-deployment steps (documented below)
- [ ] Requires data migration (documented below)

### Checklist
- [ ] Code follows naming conventions
- [ ] No hardcoded IDs or credentials
- [ ] Security review completed
- [ ] Documentation updated
```

## 6. Deployment Process

### 6.1 Deployment Methods

| Method | Use Case | Complexity |
|--------|----------|------------|
| Change Sets | Simple deployments, small teams | Low |
| Salesforce CLI | CI/CD pipelines, large teams | Medium |
| DevOps Center | Managed releases | Medium |
| Unlocked Packages | Modular development | High |
| Managed Packages | ISV distribution | High |

### 6.2 Pre-Deployment Checklist

```markdown
## Pre-Deployment Checklist

### Code Quality
- [ ] All unit tests pass
- [ ] Code coverage ≥ 75% (individual) / ≥ 85% (overall)
- [ ] No critical code analysis warnings
- [ ] Security review completed
- [ ] Peer code review approved

### Testing
- [ ] Functional testing complete in QA
- [ ] UAT sign-off received
- [ ] Regression testing complete
- [ ] Performance testing (if applicable)

### Documentation
- [ ] Release notes prepared
- [ ] User documentation updated
- [ ] Technical documentation updated
- [ ] Rollback plan documented

### Communication
- [ ] Stakeholders notified of deployment window
- [ ] Support team briefed on changes
- [ ] User training scheduled (if needed)

### Technical
- [ ] All dependencies deployed first
- [ ] Custom settings/metadata configured
- [ ] Integration endpoints verified
- [ ] Backup taken (for data changes)
```

### 6.3 Deployment Sequence

```
Standard Deployment Order:

1. FOUNDATION
   - Custom Objects
   - Custom Fields
   - Record Types
   - Page Layouts

2. CONFIGURATION
   - Custom Settings
   - Custom Metadata Types
   - Custom Labels
   - Static Resources

3. SECURITY
   - Permission Sets
   - Permission Set Groups
   - Sharing Rules

4. AUTOMATION INFRASTRUCTURE
   - Apex Classes (non-trigger)
   - Apex Interfaces
   - Invocable Actions

5. TRIGGERS
   - Apex Triggers
   - Trigger Handlers

6. FLOWS
   - Subflows first
   - Main flows second

7. UI COMPONENTS
   - Lightning Web Components
   - Aura Components
   - Visualforce Pages/Components
   - Lightning Pages

8. FINAL
   - Reports
   - Dashboards
   - Email Templates
```

### 6.4 CLI Deployment Script

```bash
#!/bin/bash
# deploy.sh - Standard deployment script

# Configuration
SOURCE_ORG="DevSandbox"
TARGET_ORG=$1
DEPLOY_DIR="force-app/main/default"

# Validate arguments
if [ -z "$TARGET_ORG" ]; then
    echo "Usage: ./deploy.sh <target-org-alias>"
    exit 1
fi

echo "=== Pre-deployment Validation ==="

# Run tests locally
echo "Running Apex tests..."
sf apex run test --target-org $SOURCE_ORG --code-coverage --result-format human

if [ $? -ne 0 ]; then
    echo "ERROR: Tests failed. Deployment aborted."
    exit 1
fi

echo "=== Starting Deployment to $TARGET_ORG ==="

# Deploy with test execution
sf project deploy start \
    --target-org $TARGET_ORG \
    --source-dir $DEPLOY_DIR \
    --test-level RunLocalTests \
    --wait 60

if [ $? -eq 0 ]; then
    echo "=== Deployment Successful ==="
else
    echo "=== Deployment Failed ==="
    exit 1
fi

echo "=== Post-deployment Verification ==="
# Add verification commands here
```

### 6.5 Post-Deployment Checklist

```markdown
## Post-Deployment Checklist

### Immediate (within 15 minutes)
- [ ] Deployment status verified as successful
- [ ] Apex tests pass in target environment
- [ ] No deployment errors in Setup > Deploy > Deployment Status

### Functional Verification (within 1 hour)
- [ ] Key user workflows tested
- [ ] Integration endpoints responding
- [ ] Reports/dashboards loading correctly
- [ ] New features functioning as expected

### Monitoring (24-48 hours)
- [ ] Error logs reviewed
- [ ] Performance metrics within acceptable range
- [ ] User feedback collected
- [ ] No unexpected behavior reported

### Documentation
- [ ] Deployment logged in release tracker
- [ ] Release notes published
- [ ] Technical documentation finalized
```

## 7. Rollback Procedures

### 7.1 Rollback Decision Matrix

| Scenario | Action |
|----------|--------|
| Deployment failed | Automatic rollback (if possible) |
| Critical bug in production | Emergency rollback + hotfix |
| Performance degradation | Assess impact, rollback if severe |
| Minor issues | Fix forward (new deployment) |

### 7.2 Rollback Methods

| Method | Speed | Complexity | Risk |
|--------|-------|------------|------|
| Redeploy previous version | Medium | Low | Low |
| Change Set rollback | Slow | Medium | Medium |
| Quick Deploy (validated) | Fast | Low | Low |
| Manual revert | Variable | High | High |

### 7.3 Rollback Procedure

```markdown
## Emergency Rollback Procedure

1. ASSESS (5 min)
   - Identify affected components
   - Determine rollback scope
   - Notify stakeholders

2. PREPARE (10 min)
   - Locate previous version in source control
   - Verify rollback package/change set
   - Confirm rollback won't cause data issues

3. EXECUTE (15-30 min)
   - Deploy rollback package
   - Monitor deployment status
   - Verify tests pass

4. VERIFY (15 min)
   - Test critical functionality
   - Confirm issue is resolved
   - Check for side effects

5. COMMUNICATE
   - Notify stakeholders of rollback
   - Update incident ticket
   - Schedule post-mortem

Total Time Target: < 1 hour
```

### 7.4 Rollback Prevention

```
Strategies to minimize rollback needs:

1. Feature Flags
   - Use Custom Metadata to enable/disable features
   - Deploy code inactive, activate when ready

2. Canary Deployments
   - Deploy to limited user group first
   - Monitor before full rollout

3. Database Rollback
   - Always backup before data migrations
   - Use reversible data operations

4. Quick Deploy
   - Validate deployments before execution
   - Use Quick Deploy for pre-validated packages
```

## 8. Release Management

### 8.1 Release Cadence

| Release Type | Frequency | Scope | Lead Time |
|--------------|-----------|-------|-----------|
| Major Release | Quarterly | Large features, architecture | 4-6 weeks |
| Minor Release | Monthly | Features, enhancements | 2-3 weeks |
| Patch Release | Weekly/Bi-weekly | Bug fixes, small changes | 1 week |
| Hotfix | As needed | Critical issues | 1-4 hours |

### 8.2 Release Planning

```markdown
## Release Plan Template

**Release:** [Name/Version]
**Target Date:** [YYYY-MM-DD]
**Release Manager:** [Name]

### Scope
| Ticket | Description | Priority | Status |
|--------|-------------|----------|--------|
| JIRA-1 | Feature A | High | Ready |
| JIRA-2 | Bug fix B | Medium | In QA |

### Timeline
- Code freeze: [Date]
- QA testing: [Start] - [End]
- UAT: [Start] - [End]
- Go/No-go decision: [Date]
- Production deployment: [Date/Time]

### Dependencies
- [ ] Dependency 1
- [ ] Dependency 2

### Risks
| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Risk 1 | Medium | High | Mitigation strategy |

### Communication Plan
- Stakeholder notification: [Date]
- User communication: [Date]
- Training: [Date]
```

### 8.3 Go/No-Go Criteria

```markdown
## Go/No-Go Checklist

### Must Have (All required for Go)
- [ ] All critical/high priority items complete
- [ ] UAT sign-off received
- [ ] Test coverage meets requirements
- [ ] No critical/high bugs open
- [ ] Rollback plan documented and tested
- [ ] All approvals received

### Should Have (Preferred but not blocking)
- [ ] All medium priority items complete
- [ ] Performance testing complete
- [ ] Documentation finalized
- [ ] Training materials ready

### Decision
- **GO**: All "Must Have" items checked
- **NO-GO**: Any "Must Have" item unchecked
- **CONDITIONAL GO**: Go with documented exceptions
```

## 9. Hotfix Process

### 9.1 Hotfix Criteria

```
A hotfix is justified when:
- Critical business process is blocked
- Security vulnerability is discovered
- Data integrity is at risk
- Significant financial impact
- Regulatory/compliance issue

A hotfix is NOT for:
- Nice-to-have features
- Minor bugs that have workarounds
- Performance issues (unless critical)
- UI/UX improvements
```

### 9.2 Hotfix Workflow

```
1. IDENTIFY (< 15 min)
   - Confirm issue severity
   - Get emergency approval

2. DEVELOP (< 2 hours)
   - Branch from main/production
   - Minimal code change
   - Write/update tests

3. TEST (< 1 hour)
   - Deploy to sandbox
   - Verify fix
   - Regression test critical paths

4. DEPLOY (< 30 min)
   - Deploy to production
   - Verify fix in production

5. DOCUMENT (< 1 hour)
   - Update ticket
   - Create post-mortem
   - Merge to all active branches
```

### 9.3 Hotfix Documentation

```markdown
## Hotfix Report

**Hotfix ID:** HF-[YYYY]-[NNN]
**Date:** [YYYY-MM-DD]
**Severity:** [Critical/High]

### Issue Description
[What was the problem?]

### Impact
- Users affected: [Number/Groups]
- Business impact: [Description]
- Duration: [Start time] - [End time]

### Root Cause
[What caused the issue?]

### Fix Description
[What was changed?]

### Verification
[How was the fix verified?]

### Prevention
[What will prevent this in the future?]

### Timeline
| Time | Action |
|------|--------|
| HH:MM | Issue reported |
| HH:MM | Investigation started |
| HH:MM | Root cause identified |
| HH:MM | Fix developed |
| HH:MM | Fix deployed |
| HH:MM | Issue resolved |
```

## 10. Compliance Checklist

Before any deployment:

- [ ] Change request documented and approved
- [ ] Code reviewed and approved
- [ ] Tests pass with required coverage
- [ ] Security review completed
- [ ] UAT sign-off received
- [ ] Rollback plan documented
- [ ] Deployment window confirmed
- [ ] Stakeholders notified
- [ ] Support team briefed
- [ ] Deployment steps documented
- [ ] Post-deployment verification planned

---

**Document Owner:** Release Management Team
**Next Review Date:** 2026-08-06
**Related Documents:**
- [Testing Standards](./testing-standards.md)
- [Security & Access Control Standard](./security-access-control.md)
