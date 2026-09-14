# 25. Merge Conflict Resolution in Release Branch

## Scenario

You're the release manager preparing to merge `release/2.0` into `main` for a production release tomorrow. The release branch has been active for 3 weeks. During this time, 4 features were merged into `release/2.0` from various feature branches. Meanwhile, 2 critical hotfixes were applied directly to `main` and cherry-picked into `release/2.0`. Now when you attempt the merge, you have conflicts in:

1. `database/migrations/0045_add_user_preferences.sql` - Two different migration files try to alter the same table
2. `config/application.yml` - Environment variables changed in both branches
3. `src/services/payment_service.py` - Logic was refactored in both branches differently
4. `docker-compose.prod.yml` - Service definitions modified in both branches

The release is tomorrow. The CEO has announced the release to investors. You cannot delay. How do you resolve these conflicts safely and ensure the release is stable?

## Interviewer Question

"You're preparing a release from release/2.0 to main and encounter complex merge conflicts involving database migrations, configuration files, and service code. The release is tomorrow and cannot be delayed. Walk me through your conflict resolution strategy."

## What I Should Think About

- Types of conflicts (content, file rename, delete/modify, add/add)
- Database migration conflict resolution strategy
- Configuration conflict resolution (environment-specific values)
- Code conflict resolution with business logic implications
- Testing strategy after conflict resolution
- Release confidence with resolved conflicts
- Who should resolve each type of conflict
- Rollback plan if resolved conflicts introduce bugs

## Ideal Answer

**Phase 1: Triage Conflicts (30 minutes)**

Before resolving anything, I categorize each conflict by type and risk:

- **Database migrations** (HIGHEST RISK) - Requires DBA review
- **Configuration files** (MEDIUM RISK) - Requires environment expert review
- **Service code** (MEDIUM RISK) - Requires feature owner review
- **Docker/infrastructure** (LOWER RISK) - Requires DevOps review

**Phase 2: Resolution Strategy by Conflict Type**

For database migrations, the rule is: **never have two migrations alter the same table in the same release.** The resolution is to combine the migrations into a single migration that applies both changes in the correct order. This requires understanding what each migration does and ensuring the combined result is correct.

For configuration files, I compare the specific values. Usually, the `main` branch has production values and the release branch has new required values. The resolution is to merge both sets of values, keeping production values from `main` where they conflict.

For service code, I review the diff from both sides, understand the intent of each change, and merge the logic. This often requires a code review, not just a text merge.

**Phase 3: Resolution and Testing**

After resolving conflicts, I run the full test suite, specifically:
1. Database migration tests (dry run the migration)
2. Integration tests for affected services
3. Smoke tests against a staging environment

**Phase 4: Release**

Merge to main, tag the release, and deploy.

## Architecture

```
Conflict Resolution Flow:

main ──────●──────●──hotfix1──●──hotfix2──●
                                        ↑
release/2.0 ──●──●──●──●──●──●──●──● (conflicts here)

Conflict Categories:
├── database/migrations/0045_* (add/add conflict - both branches added different migrations)
├── config/application.yml (content conflict - different values)
├── src/services/payment_service.py (content conflict - different refactors)
└── docker-compose.prod.yml (content conflict - different service configs)

Resolution Order:
1. Database migrations → Combine into single migration
2. Configuration → Merge values, keep production defaults
3. Service code → Review intent, merge logic
4. Infrastructure → Reconcile service definitions
```

## Investigation

1. Start the merge to see all conflicts: `git merge main --no-commit --no-ff`
2. List conflicted files: `git diff --name-only --diff-filter=U`
3. For each conflict, examine both sides:
   - `git show :1:file` (base/common ancestor)
   - `git show :2:file` (ours/release branch)
   - `git show :3:file` (theirs/main branch)
4. Check git blame for who wrote conflicting lines
5. Review PRs that introduced each change
6. Check if automated tests cover the affected code paths
7. Verify database migration dependencies

## Commands

```bash
# Step 1: Start the merge (no commit, so we can resolve conflicts)
git checkout release/2.0
git merge main --no-commit --no-ff

# Step 2: See what's conflicted
git status
# Shows conflicted files

# Step 3: List all conflicted files
git diff --name-only --diff-filter=U

# Step 4: For each conflict, examine all three versions
# Base (common ancestor), Ours (release/2.0), Theirs (main)
cat database/migrations/0045_add_user_preferences.sql
# Look for <<<<<<< and >>>>>>> markers

# Step 5: Use a merge tool for complex conflicts
git mergetool

# Step 6: After resolving each file, mark as resolved
git add database/migrations/0045_add_user_preferences.sql
git add config/application.yml
git add src/services/payment_service.py
git add docker-compose.prod.yml

# Step 7: Verify no conflicts remain
git status

# Step 8: Test the resolved code
python -m pytest tests/ -v
python manage.py migrate --check  # Verify migrations are valid
docker-compose -f docker-compose.prod.yml config  # Validate docker config

# Step 9: Complete the merge
git commit --no-edit  # Uses the default merge commit message

# Step 10: Tag the release
git tag -a v2.0.0 -m "Release 2.0.0"
git push origin release/2.0 --tags

# Step 11: Merge to main for deployment
git checkout main
git merge release/2.0 --no-ff
git push origin main
git tag -a v2.0.0-main -m "Production release 2.0.0"
git push origin v2.0.0-main

# Alternative: If you need to abort and start over
git merge --abort
```

## Root Cause

1. **Long-lived release branch** - 3 weeks is too long; conflicts accumulate
2. **Lack of regular rebasing** - Feature branches should rebase on release branch frequently
3. **Hotfixes applied to main without同步到release branch promptly**
4. **Multiple developers modifying same files** without coordination
5. **No merge conflict prevention strategy** (e.g., CODEOWNERS, file-level ownership)
6. **Database migrations not following expand-contract pattern** - causing structural conflicts
7. **Insufficient branching strategy enforcement** - unclear rules on where changes go

## Immediate Mitigation

1. **Assign conflict resolution by expertise:**
   - DBA resolves migration conflicts
   - Backend lead resolves service code conflicts
   - DevOps resolves infrastructure conflicts
   - Config owner resolves configuration conflicts

2. **Resolve in order of risk** (migrations first, then code, then config)

3. **Run comprehensive tests** after resolution:
   ```bash
   # Dry-run migration
   python manage.py migrate --plan
   # Full test suite
   pytest --tb=long
   # Integration tests
   pytest tests/integration/ -v
   ```

4. **Deploy to staging first** and verify before production

## Permanent Fix

1. **Shorten release cycles** - Move to weekly releases instead of 3-week cycles
2. **Implement merge queue** - Automate merging with conflict detection
3. **Rebase feature branches frequently** - Daily if possible
4. **Use CODEOWNERS** to prevent conflicting changes to same files
5. **Establish release branch rules:**
   - Hotfixes go to release branch first, then cherry-pick to main
   - Feature freeze after branch creation (only bug fixes)
   - Regular syncs from main to release branch

6. **Adopt trunk-based development** to eliminate long-lived release branches entirely:
   ```
   Trunk-based: main → feature flags → release
   vs
   GitFlow: main ← release/2.0 ← feature/* ← develop
   ```

## Monitoring

- Track merge conflict frequency per release
- Monitor time spent on conflict resolution
- Alert when release branches live longer than target
- Track test pass rates after conflict resolution
- Monitor deployment success rate for releases with high conflict counts

## Security

- Verify conflict resolution doesn't introduce security regressions
- Review configuration conflicts for sensitive values (API keys, secrets)
- Ensure database migration conflicts don't expose data
- Check that resolved code doesn't bypass authentication/authorization
- Audit merge commits for unexpected changes

## Production Considerations

- **Testing:** Extra thorough testing required after conflict resolution
- **Rollback:** Have a tested rollback plan before deploying
- **Staging:** Deploy to staging environment and run smoke tests
- **Communication:** Inform stakeholders of resolution timeline
- **Monitoring:** Enhanced monitoring during and after deployment
- **Feature flags:** Use feature flags for risky merged features

## Senior-Level Answer

"I'd triage conflicts by type and risk. Database migrations get DBA review and are combined into a single migration using the expand-contract pattern. Configuration conflicts keep production values from main while incorporating new values from the release. Code conflicts require understanding the intent of both changes and merging the logic, ideally with the feature owner reviewing. I'd run the full test suite, do a dry-run migration, deploy to staging, and verify before production. Long-term, I'd push for shorter release cycles and trunk-based development to prevent this situation."

## Architect-Level Answer

"This conflict situation is a symptom of a branching strategy that doesn't scale. Three-week release branches with hotfixes flowing in both directions create merge hell. The architectural solution is to adopt trunk-based development with feature flags for release gating. This eliminates long-lived release branches entirely. For the immediate situation, I'd establish a conflict resolution protocol: (1) automated conflict detection in CI, (2) CODEOWNERS for file-level ownership, (3) a merge queue that serializes merges and detects conflicts early, and (4) a release automation tool that validates the release branch against main before attempting the merge. The goal is to make conflicts impossible by design, not resolution."

## Follow-Up Questions

1. "How do you handle a conflict in a database migration when both branches added a column to the same table?"
2. "What's the difference between `git merge --strategy-option theirs` and manual conflict resolution?"
3. "How would you implement a merge queue to prevent conflicts from accumulating?"
4. "Explain the expand-contract pattern and how it prevents migration conflicts."
5. "If the release must go out tomorrow but the conflict resolution introduces a new bug, what's your fallback plan?"
