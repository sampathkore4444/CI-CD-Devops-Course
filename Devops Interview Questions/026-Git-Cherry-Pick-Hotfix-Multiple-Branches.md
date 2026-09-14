# 26. Cherry-Picking Hotfix to Multiple Branches

## Scenario

A critical CVE (CVE-2024-38014) was discovered in a third-party authentication library your application uses. The vulnerability allows remote code execution via crafted JWT tokens. Your security team rated it as CVSS 9.8 (Critical). You've already patched the vulnerability on `main` by updating the library version from `2.3.1` to `2.3.4` and updating the lock file. The fix was a single commit `f7e8d9c`. Now you need to apply this same fix to:

- `release/1.8` (uses library version 2.1.x, still needs the patch but different lock file format)
- `release/1.9` (uses library version 2.3.x, same lock file format as main)
- `release/2.0` (uses library version 2.3.x, same lock file format as main)

The `release/1.8` branch uses an older version of the dependency manager, so the lock file format is different. You also need to verify the fix doesn't break existing functionality on each branch. Patches to production environments need to go out within 24 hours per your security policy.

## Interviewer Question

"A critical security vulnerability has been fixed on main. You need to apply this fix to 3 active release branches without introducing regressions. Walk me through your cherry-pick strategy, including how to handle branch-specific differences."

## What I Should Think About

- Cherry-pick mechanics (commit hashes, parent tracking, conflicts)
- Order of cherry-picking across branches
- Handling different lock file formats across branches
- Testing each cherry-picked commit independently
- Commit message conventions for hotfixes
- Branch-specific adaptations (not all cherry-picks are clean)
- Verification strategy before deploying
- Communication and tracking across branches

## Ideal Answer

**Phase 1: Analyze the Fix (15 minutes)**

Before cherry-picking, understand what the fix actually changes. In this case, it's a dependency version bump. But because `release/1.8` uses a different version of the dependency manager, the lock file format differs. This means the cherry-pick to `release/1.8` will likely have conflicts that need manual resolution.

**Phase 2: Cherry-Pick in Order of Risk (30-60 minutes)**

Start with the cleanest branch (release/1.9 or 2.0) to verify the cherry-pick works, then handle the more complex branch (1.8).

For each branch:
1. Checkout the branch
2. Cherry-pick the commit
3. Resolve any conflicts
4. Run tests
5. Commit with a clear message referencing the original commit
6. Push and create a PR for review

**Phase 3: Verify and Deploy**

Run the CI/CD pipeline on each branch. Deploy to staging, then production.

## Architecture

```
Original Fix:
main ──────●──────●──────f7e8d9c (CVE fix)
                           │
Cherry-Pick Targets:
release/1.8 ←──────────── cherry-pick (CONFLICT - different lock file format)
release/1.9 ←──────────── cherry-pick (CLEAN - same format)
release/2.0 ←──────────── cherry-pick (CLEAN - same format)

Branch Comparison:
├── release/1.8: lib v2.1.x, pip v20.x (old lock format) → CONFLICT
├── release/1.9: lib v2.3.x, pip v23.x (new lock format) → CLEAN
└── release/2.0: lib v2.3.x, pip v23.x (new lock format) → CLEAN
```

## Investigation

1. Examine the fix commit: `git show f7e8d9c --stat`
2. Check which files were modified
3. Compare lock file formats across branches
4. Check if the library version 2.3.4 is compatible with older library versions used in release/1.8
5. Review test coverage for the authentication module on each branch
6. Check if there are any existing hotfix tracking issues
7. Verify each branch's CI/CD pipeline is functional

## Commands

```bash
# Step 1: Examine the fix commit
git show f7e8d9c
# Shows the diff, author, commit message

# Step 2: Start with release/1.9 (most likely clean)
git checkout release/1.9
git cherry-pick f7e8d9c
# If clean: proceeds automatically
# If conflicts: resolve them

# Step 3: If there are conflicts during cherry-pick
git status  # Shows conflicted files
# Manually resolve conflicts in each file
git add <resolved-files>
git cherry-pick --continue

# Step 4: Add a reference to the original commit in the message
git commit --amend  # Edit message to include:
# "(cherry picked from commit f7e8d9c)
# Fixes CVE-2024-38014
# Original-Commit-Id: f7e8d9c"

# Step 5: Push and create PR
git push origin release/1.9
gh pr create --base release/1.9 --title "Hotfix: CVE-2024-38014" \
  --body "Cherry-pick of f7e8d9c from main. Fixes CVE-2024-38014."

# Step 6: Repeat for release/2.0
git checkout release/2.0
git cherry-pick f7e8d9c
git push origin release/2.0
gh pr create --base release/2.0 --title "Hotfix: CVE-2024-38014"

# Step 7: Handle release/1.8 (likely conflicts)
git checkout release/1.8
git cherry-pick f7e8d9c
# CONFLICT in requirements.lock (different format)

# Step 8: Resolve the conflict manually
# The old lock file format needs manual editing
# Update the library version in the old format
vim requirements.lock
# Change: auth-lib==2.1.3 → auth-lib==2.1.4 (or compatible patched version)

git add requirements.lock
git cherry-pick --continue

# Step 9: Run tests on each branch
git checkout release/1.8 && python -m pytest tests/ -v
git checkout release/1.9 && python -m pytest tests/ -v
git checkout release/2.0 && python -m pytest tests/ -v

# Step 10: Track all cherry-picks
# Create a tracking issue or spreadsheet:
# Branch      | Commit    | Status    | Deployed
# release/1.8 | abc1234   | Merged    | Staging
# release/1.9 | def5678   | Merged    | Production
# release/2.0 | ghi9012   | Merged    | Production
```

## Root Cause

1. **Third-party dependency vulnerability** - No automated dependency vulnerability scanning
2. **Multiple active release branches** - Increases surface area for patches
3. **Inconsistent dependency management** - Different lock file formats across branches
4. **No automated security patching workflow** - Manual cherry-pick process is slow and error-prone
5. **Lack of dependency update automation** - Dependabot/Renovate not configured
6. **Long-lived release branches** - More branches = more cherry-picks needed

## Immediate Mitigation

1. **Apply the cherry-pick to each branch** starting with the cleanest
2. **Run tests on each branch** before merging
3. **Deploy to staging first** to verify
4. **Deploy to production** with enhanced monitoring
5. **Verify the fix** by testing JWT token handling

## Permanent Fix

1. **Automate dependency vulnerability scanning:**
   ```yaml
   # GitHub Actions workflow
   - uses: actions/dependency-review-action@v3
   - uses: github/codeql-action/analyze
   ```

2. **Set up Dependabot or Renovate** for automated dependency updates

3. **Implement security patching runbook:**
   - Document the cherry-pick process for each branch
   - Include branch-specific instructions
   - Maintain a patch tracking spreadsheet

4. **Reduce active release branches:**
   - Move to trunk-based development
   - Use feature flags for release gating
   - Only maintain 1-2 active release branches

5. **Automate cross-branch patching:**
   ```bash
   # Script to cherry-pick across all active branches
   for branch in release/1.8 release/1.9 release/2.0; do
     git checkout $branch
     git cherry-pick f7e8d9c || echo "Conflict on $branch - needs manual resolution"
     git push origin $branch
   done
   ```

## Monitoring

- Alert on new CVEs in dependencies (via Dependabot/Snyk/Trivy)
- Track cherry-pick success rate across branches
- Monitor deployment status for each patched branch
- Verify fix effectiveness through security testing
- Track time from vulnerability discovery to deployment

## Security

- Verify the fix actually addresses CVE-2024-38014
- Test for the specific vulnerability after deployment
- Review the library's changelog for any other security fixes in 2.3.4
- Check if the vulnerability was exploited before the fix
- Audit access logs for suspicious JWT token activity
- Consider revoking existing JWT tokens after patching

## Production Considerations

- **Blast radius:** The vulnerability affects authentication - potential for full compromise
- **Deployment order:** Patch most critical production systems first
- **Rollback:** Have rollback plan if patch breaks authentication
- **Communication:** Notify security team and stakeholders of patch status
- **Compliance:** Document patch timeline for audit requirements
- **Downtime:** Zero-downtime deployment for security patches is critical

## Senior-Level Answer

"I'd cherry-pick commit f7e8d9c to each release branch, starting with the cleanest (release/1.9 and 2.0 which share the same lock file format). For release/1.8 with the different lock file format, I'd manually resolve the conflict by updating the library version in the old format. Each cherry-pick gets its own PR with CI tests, and I'd deploy to staging before production. The commit messages reference the original commit for traceability. For the long-term, I'd set up Dependabot for automated dependency updates and reduce active release branches to minimize the patching surface area."

## Architect-Level Answer

"This situation highlights a systemic problem: maintaining multiple long-lived release branches creates a security risk because patches must be applied to each branch individually. The architectural solution is to adopt trunk-based development where there's only one main branch, and releases are cut from it with feature flags. This reduces the patching surface to one branch. For the immediate fix, I'd also implement: (1) automated dependency scanning in CI to catch vulnerabilities before merge, (2) a cross-branch patching automation tool, (3) a security response runbook with pre-defined procedures, and (4) a dependency update policy that ensures all branches stay on supported versions. The goal is to make security patching a one-branch operation, not a multi-branch coordination exercise."

## Follow-Up Questions

1. "What's the difference between cherry-pick and merge, and when would you choose one over the other?"
2. "How do you handle a cherry-pick that introduces a regression on the target branch?"
3. "How would you automate cherry-picking to multiple branches with conflict detection?"
4. "What is the `-x` flag in cherry-pick and why is it important for audit trails?"
5. "If release/1.8 uses a completely different authentication library, how would you apply the security fix?"
