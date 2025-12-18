# PR Automation Workflow Test Results

## Summary

Tested the "ready-when-green" PR automation workflow in the example repository. The test revealed several implementation issues that have been fixed in this repo and should be applied to the portal repository.

## Test Setup

- **Test Repository**: https://github.com/justincampbell/example
- **Test PR**: https://github.com/justincampbell/example/pull/1
- **Workflows**:
  - `.github/workflows/pr-automation.yml` - Main automation workflow
  - `.github/workflows/ci.yml` - Simple CI check for testing

## Issues Discovered and Fixed

### 1. Check Run Name Mismatch
**Problem**: Workflow used `github.job` to filter itself out from pending checks, but this resolves to "ready-when-green" while the actual check run name is "Ready When Green" (with capitals and spaces).

**Fix**: Hardcode the correct check run name:
```yaml
current_job_name="Ready When Green"
```

**Impact**: Without this fix, the workflow sees itself as a pending check and waits forever.

### 2. External CI Status Blocking
**Problem**: When there are no external CI systems (like Circle CI), the GitHub status API returns `state: "pending"` with zero statuses. The workflow was blocking on this even though all GitHub Actions checks had passed.

**Fix**: Only check external status if there are actual external statuses:
```yaml
if [ "$total_statuses" -gt 0 ] && [ "$status_state" != "success" ]; then
  echo "External CI status is '$status_state' (not 'success') - waiting"
  continue
fi
```

**Impact**: Without this fix, repos with only GitHub Actions (no Circle CI) would never mark PRs ready.

### 3. Draft PR Conversion API
**Problem**: Attempted to use REST API `PATCH /pulls/{pr_number}` with `draft=false`, but this doesn't work for converting drafts to ready status.

**Fix Attempted**: Use GraphQL mutation `markPullRequestReadyForReview`

**Result**: GraphQL mutation failed with "Resource not accessible by integration" error despite having `pull-requests: write` permission.

**Root Cause**: The `GITHUB_TOKEN` provided by GitHub Actions has limitations on certain operations. The `markPullRequestReadyForReview` mutation appears to require additional permissions or a different authentication approach.

## Current Status

### What Works
- Workflow correctly triggers on `pull_request` labeled events
- Workflow correctly detects draft status and presence of label
- Workflow correctly filters out itself from check count
- Workflow correctly skips external CI check when none exists
- Workflow correctly identifies when all checks have passed

### What Doesn't Work
- **Converting draft to ready**: GraphQL mutation fails with permission error

### Remaining Issue

The core functionality works, but the final step (marking PR as ready) fails due to GitHub token permissions. This is a known limitation of `GITHUB_TOKEN`:

From GitHub docs: "The GITHUB_TOKEN has certain limitations for some operations, particularly around pull requests from forks and certain GraphQL mutations."

### Possible Solutions

1. **Use a GitHub App**: Create a GitHub App with appropriate permissions and use its token
2. **Use a Personal Access Token**: Store a PAT with `repo` scope as a repository secret
3. **Different Workflow Pattern**: Instead of marking ready directly, comment on the PR to notify maintainers
4. **Wait for GitHub**: This might be a temporary limitation that GitHub addresses

## Recommendations for Portal

1. **Apply the two working fixes immediately**:
   - Fix check run name filtering
   - Fix external CI status logic

2. **For the draft conversion issue**, consider alternatives:
   - Add a comment to the PR: "All checks passed! Ready to mark for review."
   - Send a Slack notification
   - Use a GitHub App if the team wants to invest in that infrastructure

## Test Files Location

All test files are in: `~/Code/justincampbell/example`

- Workflows: `.github/workflows/`
- Documentation: `PR_AUTOMATION_TEST.md`, `TEST_RESULTS.md`
- Test PR: https://github.com/justincampbell/example/pull/1 (still draft due to permission issue)

## Next Steps

1. Apply the two working fixes to portal's pr-automation.yml workflow
2. Decide on approach for draft-to-ready conversion:
   - Option A: Use a GitHub App (most robust)
   - Option B: Use a PAT (simpler, but less secure)
   - Option C: Change to notification-only (safest, but requires manual action)

## Commit History

All fixes and iterations are documented in commits:
- Initial setup: `a74b6f1`
- Check name fix: `5d8b743`
- External CI fix: `b741c86`
- GraphQL attempt: `e203b66`
