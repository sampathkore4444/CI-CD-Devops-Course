# 23. Developer Accidentally Merged Broken Code to Production

## Scenario

It's 9:15 AM on a Monday. Your team follows a GitFlow branching model. A developer named Alex was working on a feature branch `feature/payment-refactor` and accidentally merged it directly into `main` instead of `develop`. The CI/CD pipeline (GitHub Actions) triggered automatically, built the artifact, ran integration tests (which had a flaky test that passed), and deployed to production within 12 minutes. Users are now seeing HTTP 500 errors on the checkout page. Revenue is being lost at approximately $5,000/minute. The database migration included in the merge altered the `orders` table schema by dropping a column that the legacy billing service still depends on. You have 3 other developers who have pulled `main` locally. The release/1.9 branch was branched off main last week and is also affected.

## Interviewer Question

"A developer accidentally merged broken code directly into main and it's now live in production causing errors. Walk me through your incident response, the rollback strategy, and the permanent prevention measures. Consider git revert vs git reset, database state, and team-wide impact."

## What I Should Think About

- Incident severity and blast radius assessment
- Immediate user impact vs long-term code correctness
- Git revert vs git reset vs rebase implications for shared branches
- Database migration rollback (schema changes are not trivially reversible)
- CI/CD pipeline behavior during rollback
- Communication with stakeholders
- Team members with local copies of corrupted main
- Release branch contamination
- Post-incident process improvements

## Ideal Answer

**Phase 1: Immediate Incident Response (First 5 minutes)**

First, I assess the blast radius. The checkout page is down, so this is revenue-critical. I communicate in the incident Slack channel: "P1 incident - Checkout page broken - investigating." Then I assess what exactly broke. In this case, a database column was dropped AND application code was changed.

**Phase 2: Rollback Decision (5-15 minutes)**

This is where it gets nuanced. We cannot simply `git revert` the merge commit because the database migration dropped a column. A revert would restore the application code that expects the column, but the column is still gone. We need a coordinated rollback:

1. First, restore the database column (if possible without data loss) or restore from a point-in-time backup
2. Then revert the git changes
3. Redeploy the reverted code

If the database migration is forward-only (additive), a simple `git revert` of the merge commit would work. But since a column was dropped, we need to handle DB state first.

**Phase 3: Execution**

```bash
# Find the merge commit that broke main
git log --oneline --merges -10 main

# Revert the entire merge (preserving history, safe for shared branches)
git revert -m 1 <merge-commit-hash>
git push origin main
```

The `-m 1` flag tells git to revert to the parent of the first parent (main before the merge). This is critical for merge commits.

**Phase 4: Team Coordination**

Notify all developers who pulled main to reset their local copies:

```bash
git fetch origin
git reset --hard origin/main
```

For the release/1.9 branch, cherry-pick the revert commit so it's also fixed there.

## Architecture

```
Before Incident:
main ──────●──────●──────●──────● (current, broken)
            \              ↑
             feature/payment-refactor (merged by accident)

After Rollback:
main ──────●──────●──────●──────●──────● (reverted)
                                        ↑ new commit (revert)

Branches affected:
├── main (broken, then reverted)
├── release/1.9 (contains broken code if branched after merge)
├── feature/* (developers may have pulled broken main)
└── develop (should NOT be affected in GitFlow)
```

## Investigation

1. Identify the merge commit: `git log --oneline --merges -10 main`
2. Review what changed: `git diff <parent-before-merge>..main --stat`
3. Check database migration files: `git diff <parent-before-merge>..main -- migrations/`
4. Verify CI/CD pipeline ran: Check GitHub Actions / Jenkins logs
5. Check which tests passed/failed during the auto-deploy
6. Assess database state: Run `SHOW CREATE TABLE orders;` to confirm column status
7. Check monitoring dashboards for error rates and timing correlation
8. Identify all branches containing the bad commit: `git branch -a --contains <commit>`

## Commands

```bash
# Step 1: Identify the problematic merge commit
git log --oneline --merges -10 main
# Example output:
# a1b2c3d Merge branch 'feature/payment-refactor' into main

# Step 2: See what files were affected
git diff a1b2c3d^1..a1b2c3d --stat

# Step 3: Check if database migration is included
git diff a1b2c3d^1..a1b2c3d -- "**/*migration*"

# Step 4: Revert the merge commit (safe for shared branches)
# -m 1 specifies reverting to the first parent (main before merge)
git revert -m 1 a1b2c3d --no-edit

# Step 5: Push the revert
git push origin main

# Step 6: Force CI/CD to redeploy
# (Usually automatic on push to main, or trigger manually)
gh workflow run deploy.yml --ref main

# Step 7: Notify team to sync their local main
# They should run:
git fetch origin
git reset --hard origin/main

# Step 8: If release/1.9 also needs the fix
git checkout release/1.9
git cherry-pick <revert-commit-hash>
git push origin release/1.9

# Step 9: Verify the fix
git log --oneline -5 main
# Should show the revert as the latest commit
```

## Root Cause

1. **No branch protection rules on main** - Direct push was allowed without PR review
2. **No required status checks** - CI tests didn't gate the merge
3. **Developer error** - Merged to wrong branch (main instead of develop)
4. **Lack of branch naming conventions or merge automation** - Manual merge process is error-prone
5. **Flaky integration tests** - The test that should have caught the issue passed intermittently
6. **Database migration included in feature branch** - Schema changes should be handled separately or with backward compatibility

## Immediate Mitigation

1. **Revert the merge commit on main** (using `git revert -m 1`)
2. **Restore database state** - Either re-add the dropped column or restore from point-in-time backup
3. **Trigger redeployment** of the reverted code
4. **Communicate status** to stakeholders and users
5. **Verify production health** after deployment completes

## Permanent Fix

1. **Enable branch protection on main:**
   - Require pull request reviews (minimum 2 reviewers)
   - Require status checks to pass before merging
   - Require branches to be up to date before merging
   - Restrict who can push to main (no direct pushes)

2. **Implement merge guards:**
   ```bash
   # Git hook to prevent direct pushes to main
   # .git/hooks/pre-push (on server or via GitHub settings)
   ```

3. **Separate database migrations from application code:**
   - Use a migration-first deployment strategy
   - Never drop columns in the same release as code changes
   - Implement the expand-contract pattern for schema changes

4. **Fix flaky tests** - Invest in test reliability

5. **Add merge automation:**
   - Auto-merge develop into feature branches
   - Use merge queues to prevent race conditions

## Monitoring

- Set up alerts on HTTP 500 rate spikes
- Monitor checkout page success rate (business metric)
- Track deployment success/failure rates
- Alert on unusual git activity (direct pushes to main)
- Monitor CI/CD pipeline test pass rates and flakiness

## Security

- Branch protection prevents unauthorized code changes
- Audit log review for who bypassed protections
- Consider signed commits to verify authorship
- Enforce linear history (no force pushes to main)
- CODEOWNERS file for critical paths (migrations, auth code)

## Production Considerations

- **HA:** Have a known-good deployment artifact ready for rapid rollback
- **Rollback time:** Git revert + CI/CD deploy should complete in <15 minutes
- **Database:** Always make schema changes backward-compatible (expand-contract pattern)
- **Communication:** Pre-defined incident runbook with escalation paths
- **Cost:** Each minute of downtime costs $5,000 - invest in prevention
- **Compliance:** Maintain audit trail of all changes to production (git history helps here)

## Senior-Level Answer

"I'd immediately revert the merge commit using `git revert -m 1` since it's safe for shared branches. However, since the migration dropped a database column, I'd coordinate with the DBA to restore the column first, then deploy the reverted code. For permanent prevention, I'd enforce branch protection rules requiring PR reviews, status checks, and restricted push access. I'd also implement the expand-contract pattern for database migrations so schema changes are always backward-compatible, eliminating the coupling between git rollback and database rollback."

## Architect-Level Answer

"This incident reveals three systemic issues: process, technical, and cultural. Process-wise, we need mandatory branch protection with required reviews and checks - no exceptions, even for seniors. Technically, we need to decouple database migrations from application deployments using an expand-contract pattern and feature flags. Culturally, we need blameless post-mortems and potentially pair programming for high-risk changes. I'd also implement a deployment pipeline with canary releases so broken code hits 5% of traffic before full rollout, giving us time to catch issues. The long-term architecture should support blue-green or canary deployments with automatic rollback on error rate spikes."

## Follow-Up Questions

1. "What's the difference between `git revert` and `git reset` in this context, and why would one be dangerous for shared branches?"
2. "How would you handle this if the migration had added a new column instead of dropping one?"
3. "How would you design a deployment pipeline that could automatically detect and rollback a bad deployment?"
4. "What is the expand-contract migration pattern, and how does it prevent this class of incident?"
5. "If 5 developers have local commits on top of the broken main, how do you recover their work while fixing the branch?"
