# 28. Protected Branch Bypass - Process Failure

## Scenario

Your organization has strict branch protection rules on `main`: minimum 2 PR reviews, all status checks must pass, and no direct pushes. These rules were implemented after a previous incident. However, last night at 11:30 PM, a senior developer named Sarah used her GitHub admin privileges to temporarily bypass these protections and pushed a "quick fix" directly to `main`. The commit changed 3 files in the authentication module:

1. `src/auth/jwt_handler.py` - Modified token validation logic
2. `src/auth/middleware.py` - Changed session timeout behavior
3. `config/auth.yml` - Updated rate limiting configuration

Sarah didn't run the full test suite. She only ran a subset of tests locally. The change was deployed automatically by the CI/CD pipeline. At 1:45 AM, production error rates spiked by 400%. Users were getting logged out every 5 minutes, and some could not log in at all. The on-call engineer noticed at 2:15 AM and paged you. Sarah is now unreachable (phone off). You need to investigate, rollback, and implement measures to prevent any admin from bypassing branch protection again.

## Interviewer Question

"A senior developer bypassed protected branch rules using admin privileges, pushed directly to main, and introduced a bug that took down production. Walk me through your immediate response, the investigation, and the permanent process improvements to prevent this from recurring."

## What I Should Think About

- Incident response and severity assessment
- Git history forensics (who, what, when, how)
- Rollback strategy (revert vs reset)
- Admin privilege audit and removal
- Branch protection rule hardening
- Cultural and process changes
- GitHub/GitLab admin vs maintainer roles
- Audit logging and monitoring
- Team accountability without blame culture

## Ideal Answer

**Phase 1: Immediate Incident Response (First 15 minutes)**

1. Page the incident commander
2. Start an incident channel
3. Begin rollback of the problematic commit
4. Communicate to stakeholders

**Phase 2: Rollback (15-30 minutes)**

```bash
# Find the problematic commit
git log --oneline --author="Sarah" -5 main

# Revert it
git revert <commit-hash>
git push origin main
```

**Phase 3: Investigation (30-60 minutes)**

Review the GitHub audit log to understand:
- When the protection was bypassed
- How it was done (admin API, web UI, API token)
- What was changed

**Phase 4: Permanent Prevention (1-2 weeks)**

1. Remove admin privileges from developers
2. Implement CODEOWNERS
3. Enable audit logging with alerts
4. Create a process for emergency bypasses

## Architecture

```
Before Incident:
main ──●──●──●──● (protected: 2 reviews + CI checks)
                            ↑
                   Sarah bypassed using admin privileges

After Incident:
main ──●──●──●──●──● (reverted commit)
                     ↑
              Revert commit

Process Improvement:
├── Remove admin privileges from all developers
├── Implement emergency bypass procedure
├── Add audit logging alerts
├── Create CODEOWNERS file
└── Implement merge queue
```

## Investigation

1. Check GitHub audit log for the bypass event
2. Identify the exact commit and what changed
3. Review the CI/CD pipeline logs for the auto-deploy
4. Check error monitoring for the production impact
5. Review the authentication module test coverage
6. Identify what tests Sarah ran locally vs what should have been run
7. Check if other admins have bypassed protections before

## Commands

```bash
# Step 1: Find the problematic commit
git log --oneline --all --since="2024-01-15 23:00" --until="2024-01-16 02:00"
# Or search by author
git log --oneline --author="sarah" -10 main

# Step 2: See what changed
git show <commit-hash> --stat
git show <commit-hash> -- src/auth/

# Step 3: Revert the commit
git revert <commit-hash> --no-edit
git push origin main

# Step 4: Verify the revert
git log --oneline -3 main
# Should show revert as latest

# Step 5: Check GitHub audit log via API
gh api orgs/{org}/audit-log --jq '.[] | select(.action == "repo.destroy" or .action == "protected_branch.override")'

# Step 6: Review branch protection status
gh api repos/{org}/{repo}/branches/main/protection

# Step 7: Check who has admin access
gh api repos/{org}/{repo}/collaborators --jq '.[] | select(.role_name == "admin")'

# Step 8: List all direct pushes to main (should be zero)
git log --oneline --no-merges main | head -20
# All entries should be merge commits, not direct pushes

# Step 9: Create a CODEOWNERS file
cat > .github/CODEOWNERS << 'EOF'
# Authentication module - require senior dev + security review
/src/auth/ @senior-devs @security-team
/config/auth.yml @senior-devs @security-team

# Database migrations - require DBA review
/database/ @dba-team

# Infrastructure - require DevOps review
/infrastructure/ @devops-team
docker-compose*.yml @devops-team
EOF

git add .github/CODEOWNERS
git commit -m "chore: add CODEOWNERS for critical paths"
git push origin main

# Step 10: Enable branch protection with strict settings
# (via GitHub UI or API)
gh api repos/{org}/{repo}/branches/main/protection -X PUT -f '{
  "required_status_checks": {
    "strict": true,
    "contexts": ["ci/build", "ci/test", "ci/security-scan"]
  },
  "required_pull_request_reviews": {
    "required_approving_review_count": 2,
    "dismiss_stale_reviews": true,
    "require_code_owner_reviews": true
  },
  "enforce_admins": true,
  "restrictions": null
}'
```

## Root Cause

1. **Admin privileges not restricted** - Sarah had admin access to bypass protections
2. **No enforcement of admin restrictions** - GitHub allows admins to bypass by default
3. **"Emergency" bypass without process** - No formal emergency override procedure
4. **Lack of audit monitoring** - No alerts on protection bypass events
5. **Cultural issue** - Seniority used to skip process
6. **Insufficient test coverage** - Sarah's local tests didn't catch the bug
7. **Automatic deployment on main** - No manual gate for emergency pushes

## Immediate Mitigation

1. **Revert the problematic commit** on main
2. **Verify production health** after revert deployment
3. **Enable "Include administrators"** in branch protection settings
4. **Audit all admin accounts** and remove unnecessary admin privileges
5. **Review recent commits** to main for any other bypasses
6. **Communicate incident** to the team

## Permanent Fix

1. **Remove admin privileges from all developers:**
   - Only DevOps/Platform team should have admin access
   - Developers should have write or maintain access at most

2. **Enable "Include administrators" in branch protection:**
   ```json
   {
     "enforce_admins": true,
     "required_pull_request_reviews": {
       "required_approving_review_count": 2,
       "dismiss_stale_reviews": true,
       "require_code_owner_reviews": true
     }
   }
   ```

3. **Create an emergency bypass procedure:**
   ```
   Emergency Bypass Procedure:
   1. Post in #incident channel with justification
   2. Get approval from on-call manager
   3. DevOps temporarily disables protection (with audit log entry)
   4. Push the change
   5. DevOps re-enables protection within 1 hour
   6. Create a follow-up PR for proper review
   7. Post-incident review within 24 hours
   ```

4. **Implement audit monitoring:**
   ```yaml
   # Alert on branch protection bypass events
   - alert: BranchProtectionBypassed
     expr: github_audit_log_action{action="protected_branch.override"} > 0
     for: 0m
     labels:
       severity: critical
   ```

5. **Add CODEOWNERS** for critical code paths

6. **Implement merge queue** to prevent race conditions

## Monitoring

- Alert on any branch protection bypass events
- Monitor for direct pushes to protected branches
- Track admin privilege usage
- Alert on authentication error rate spikes
- Monitor deployment success/failure rates
- Track PR review compliance

## Security

- Admin privilege audit is a security requirement
- Branch protection bypass is a security event
- Authentication module changes require security review
- Audit log retention for compliance
- Consider SOC 2 implications of protection bypasses

## Production Considerations

- **Incident response time:** 2:15 AM detection means 2+ hours of degraded service
- **Rollback time:** Should be <15 minutes with proper tooling
- **Process compliance:** Emergency procedures must be documented and tested
- **Team culture:** Blameless post-mortem, but clear accountability
- **Compliance:** Regulatory requirements may mandate protection rules

## Senior-Level Answer

"I'd immediately revert the commit and then enable 'enforce_admins' in branch protection settings so no one can bypass, including admins. I'd audit all admin accounts and remove unnecessary privileges. For the process, I'd create a formal emergency bypass procedure that requires documented approval, time-limited protection disabling, and mandatory follow-up PRs. I'd also add CODEOWNERS for critical paths and set up audit log alerts for any protection bypass attempts."

## Architect-Level Answer

"This incident reveals a governance gap that goes beyond Git configuration. The architectural response involves: (1) Implementing the principle of least privilege - developers should never have admin access to production repositories, (2) Creating an auditable emergency bypass process with time limits and mandatory documentation, (3) Implementing defense in depth with CODEOWNERS, merge queues, and CI/CD gates, (4) Establishing a culture where process compliance is valued over speed, and (5) Building automated monitoring that detects and alerts on policy violations in real-time. The key insight is that branch protection rules are only as strong as the enforcement mechanism - if admins can bypass, the rules are advisory, not mandatory. The solution is both technical (enforce_admins=true) and cultural (making bypass a formal, audited process)."

## Follow-Up Questions

1. "What's the difference between admin, maintain, and write access in GitHub, and how do they relate to branch protection?"
2. "How would you implement a time-limited emergency bypass that automatically re-enables protection?"
3. "How do you handle a situation where a developer claims the bypass was justified but the post-mortem disagrees?"
4. "What is a merge queue and how does it complement branch protection rules?"
5. "How would you audit the effectiveness of your branch protection rules across all repositories in an organization?"
