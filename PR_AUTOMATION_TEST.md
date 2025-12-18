# PR Automation Workflow Test

This repository serves as a test environment for the "ready-when-green" PR automation workflow.

## What It Does

The workflow automatically marks draft PRs as "ready for review" when:
1. The PR has the `ready-when-green` label
2. All CI checks have passed

## How It Works

1. **Workflow Triggers**: The automation listens for:
   - `check_suite` completion events (when GitHub Actions finish)
   - `status` events (for external CI systems)
   - `pull_request` labeled events (immediate check when label is added)

2. **Workflow Logic**:
   - Finds PRs associated with the commit
   - Checks if PR is a draft and has the `ready-when-green` label
   - Verifies all CI checks have passed
   - If all conditions met: marks PR as ready and removes the label

3. **Safety Features**:
   - Won't mark ready if no checks exist (prevents accidents)
   - Won't mark ready if any checks are pending or failed
   - Excludes itself from check validation (prevents circular dependency)

## Test Setup

### Workflows

1. **`.github/workflows/pr-automation.yml`**: The main automation workflow
   - Generic version (no project-specific references)
   - Handles check completion and label events
   - Marks draft PRs as ready when green

2. **`.github/workflows/ci.yml`**: Minimal CI check for testing
   - Runs a simple test that takes ~5 seconds
   - Provides the "green check" we're waiting for

### Testing the Workflow

**IMPORTANT**: Workflows must be on the `main` branch before they can respond to events.

#### Step 1: Merge Workflows to Main
```bash
git add .github/workflows/
git commit -m "Add PR automation and CI workflows"
git push origin main
```

#### Step 2: Create Test Branch and Draft PR
```bash
# Create test branch
git checkout -b test-pr-automation

# Make a change
echo "Test change" >> README.md
git add README.md
git commit -m "Test PR automation workflow"
git push origin test-pr-automation

# Create draft PR via GitHub CLI
gh pr create \
  --draft \
  --title "Test: PR Automation Workflow" \
  --body "This draft PR tests the ready-when-green automation. The CI check should pass, and this PR should automatically be marked as ready for review."
```

#### Step 3: Add the Label
```bash
gh pr edit --add-label "ready-when-green"
```

#### Step 4: Observe the Magic
- The CI workflow will run and pass
- When the check completes, the `pr-automation` workflow will trigger
- The workflow will detect the draft PR with the label
- It will verify all checks passed
- It will mark the PR as ready for review and remove the label

### Manual Testing

You can test different scenarios:

1. **Label before checks complete**: Add label immediately - workflow waits for checks
2. **Label after checks pass**: Add label after CI passes - workflow triggers immediately
3. **Failed checks**: Make CI fail - workflow won't mark ready
4. **Non-draft PR**: Remove draft status first - workflow skips it

### Viewing Workflow Runs

```bash
# View workflow runs
gh run list

# View specific run logs
gh run view <run-id> --log
```

## Expected Behavior

When everything is working correctly:

1. Create draft PR with changes
2. Add `ready-when-green` label
3. CI check runs and passes (~5 seconds)
4. PR automation workflow triggers automatically
5. Workflow logs show:
   - Finding the PR
   - Checking it's a draft with the label
   - Verifying CI passed
   - Marking as ready for review
   - Removing the label
6. PR is now ready for review (no longer draft)

## Troubleshooting

- **Workflow doesn't trigger**: Ensure workflows are on the `main` branch
- **Can't find PR**: Check that the commit SHA matches between branch and PR
- **No checks found**: CI workflow must run first (triggered by push to PR branch)
- **Workflow skips PR**: Verify PR is draft and has the exact label name `ready-when-green`

## References

- Original workflow source: Huntress Portal repository
- Test repository: https://github.com/justincampbell/example
