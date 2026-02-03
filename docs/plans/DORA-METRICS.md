# DORA Metrics Implementation Plan

**Status:** Not Started
**Scope:** Organization-level (medica-flow)
**Priority:** TBD

## Overview

Implement DORA (DevOps Research and Assessment) metrics tracking across all repos in the medica-flow organization.

## Metrics to Track

### 1. Deployment Frequency
How often code is deployed to production.

### 2. Lead Time for Changes
Time from commit to production deployment.

### 3. Change Failure Rate
Percentage of deployments that cause failures or require rollback.

### 4. Mean Time to Recovery (MTTR)
Time to recover from a deployment failure.

## Open Questions

- [ ] Where should metrics be stored/visualized?
  - GitHub's built-in Deployments API (free, visible in repo insights)
  - External dashboard (Datadog, Grafana, etc.)
  - Both

- [ ] What counts as a "deployment"?
  - Each environment (dev, qa, prod) separately?
  - Only production?

- [ ] How to track failures?
  - Workflow failure = deployment failure?
  - Manual marking (e.g., via PR label or issue)?

- [ ] Existing observability stack to integrate with?

## Proposed Implementation

### Phase 1: GitHub Deployments API
Add deployment tracking to terraform-ci reusable workflows:

1. Create GitHub Deployment on workflow start
2. Update deployment status on success/failure
3. Automatically captures:
   - Deployment frequency (deployments per repo/org)
   - Lead time (commit SHA → deployment timestamp)
   - Success/failure status

### Phase 2: Failure Tracking
- Track rollbacks via reverted commits or manual tags
- Calculate change failure rate from deployment statuses

### Phase 3: MTTR Tracking
- Track time between failed deployment and next successful deployment
- Optionally integrate with incident management (GitHub Issues, PagerDuty, etc.)

### Phase 4: Dashboard (Optional)
- Aggregate metrics across org
- Historical trends
- Team-level breakdowns

## Files to Modify

- `terraform-ci/.github/workflows/terraform-apply.yml` - Add deployment tracking
- `terraform-ci/.github/workflows/terraform-plan.yml` - Optionally track PR deployments
- New: `terraform-ci/.github/actions/track-deployment/action.yml` - Reusable action for deployment tracking

## References

- [GitHub Deployments API](https://docs.github.com/en/rest/deployments)
- [DORA Metrics](https://cloud.google.com/blog/products/devops-sre/using-the-four-keys-to-measure-your-devops-performance)
