# 24. Git Repository Corruption - Recovery Required

## Scenario

It's Wednesday afternoon. Your team's primary application repository on GitHub Enterprise suddenly returns errors when engineers try to push or pull. The web interface shows "repositoryCorrupted" errors. `git fetch` and `git clone` operations fail with `error: object file is empty` and `fatal: packedGE*aintegrity check failed`. You have 4 local clones across different machines (your workstation, a build server, a staging server, and a developer's laptop who's been offline for 2 days). The corrupted repository contains 5 years of history with 50,000+ commits. CI/CD pipelines are broken because they can't pull. You need to recover without losing any commits. The repo is 2GB in size.

## Interviewer Question

"A critical Git repository has become corrupted and engineers cannot clone or pull. You have local clones on several machines. Walk me through your recovery strategy to restore the repository without losing any commits."

## What I Should Think About

- Types of git corruption (object corruption, pack file corruption, reference corruption)
- Which local clone is most likely to have good state
- Verification strategy before pushing recovered data
- Impact on open PRs and branches
- CI/CD pipeline recovery
- Team communication during recovery
- Prevention measures (mirrors, backups)
- Recovery time objectives

## Ideal Answer

**Phase 1: Assess the Corruption (5-10 minutes)**

Determine the type and extent of corruption. Check if it's object-level, pack-file level, or reference-level corruption. Try `git fsck --full` on each local clone to identify which ones have intact object stores.

**Phase 2: Find the Healthiest Clone (10-20 minutes)**

```bash
# On each local clone, run full filesystem check
git fsck --full --no-dangling 2>&1 | head -50
```

The clone with zero errors or the fewest errors is your recovery source. The developer's laptop clone (2 days old) might be the cleanest if the corruption happened recently.

**Phase 3: Verify and Recover (20-60 minutes)**

Use the healthiest clone as the source. Push all branches and tags from that clone to a new bare repository, then replace the remote.

```bash
# Create a bare repository from the healthiest clone
git clone --bare /path/to/healthy/clone /tmp/recovery.git

# Verify integrity
cd /tmp/recovery.git
git fsck --full

# Push to a new GitHub repo or re-initialize the existing one
git push --mirror git@github.com:org/recovered-repo.git
```

**Phase 4: Team Recovery**

Have all team members re-clone or reset their local copies from the restored remote.

## Architecture

```
Recovery Strategy:

Corrupted Remote (GitHub)
├── object corruption detected
├── clone/push/pull failing
└── status: UNUSABLE

Local Clones (Recovery Sources):
├── Workstation Clone ──→ git fsck: 3 errors
├── Build Server Clone ──→ git fsck: 0 errors  ← BEST SOURCE
├── Staging Server Clone ──→ git fsck: 1 error
└── Dev Laptop Clone (2 days old) ──→ git fsck: 0 errors

Recovery Flow:
Healthy Clone → Bare Repo → Push --mirror → New Remote → Team Re-clones
```

## Investigation

1. Check remote repo status via web interface for error messages
2. Run `git fsck --full` on each local clone to assess corruption extent
3. Check when the corruption started (correlate with recent force pushes or server issues)
4. Determine if any open PRs exist that might need to be preserved
5. Check GitHub enterprise logs for server-side issues
6. Verify which branches and tags exist in each clone
7. Compare commit counts across clones to find the most complete one
8. Test `git fetch` and `git clone` from the remote to confirm the error

## Commands

```bash
# Step 1: Diagnose corruption on remote
git clone git@github.com:org/app.git /tmp/test-clone 2>&1
# Expected error: error: object file is empty / integrity check failed

# Step 2: Check each local clone for integrity
# Run on workstation
git fsck --full --no-dangling 2>&1

# Run on build server
ssh build-server "cd /repos/app && git fsck --full --no-dangling 2>&1"

# Run on staging server
ssh staging "cd /repos/app && git fsck --full --no-dangling 2>&1"

# Run on developer laptop
ssh dev-laptop "cd ~/projects/app && git fsck --full --no-dangling 2>&1"

# Step 3: Get commit count from each clone for comparison
git rev-list --count HEAD
# Compare across all clones - the highest count is likely most complete

# Step 4: List all branches in the healthiest clone
git branch -a

# Step 5: Create a bare repository from the healthy clone
git clone --bare /path/to/healthy/clone /tmp/recovery-bare.git

# Step 6: Verify the bare repo
cd /tmp/recovery-bare.git
git fsck --full

# Step 7: Push to a new temporary repo on GitHub
git remote add recovery git@github.com:org/app-recovery.git
git push recovery --all
git push recovery --tags

# Step 8: If the original repo can be reinitialized (GitHub support)
# Delete and recreate the repo, then push
git remote set-url origin git@github.com:org/app.git
git push origin --all
git push origin --tags

# Step 9: Verify recovery
git clone git@github.com:org/app.git /tmp/verify-clone
cd /tmp/verify-clone
git fsck --full
git log --oneline -10
git branch -a | wc -l  # Should match expected branch count

# Step 10: Force all team members to re-clone
echo "Repository recovered. Please re-clone:"
echo "  git clone git@github.com:org/app.git"
echo "OR reset your local copy:"
echo "  git fetch origin"
echo "  git reset --hard origin/main"
echo "  git clean -fd"
echo "  git branch -a | xargs -I {} git branch --track {} origin/{}"
```

## Root Cause

1. **Server-side storage failure** - Disk corruption on the Git server
2. **Improper server shutdown** during write operations
3. **Pack file corruption** during aggressive garbage collection
4. **Network interruption** during push operations causing partial writes
5. **Missing repository backup strategy** - No mirror or backup existed
6. **No monitoring** on repository health (git fsck not scheduled)
7. **Single point of failure** - Only one remote, no mirrors

## Immediate Mitigation

1. **Stop all CI/CD pipelines** that are trying to pull from the corrupted repo
2. **Notify the team** not to push to or pull from the repo
3. **Identify the healthiest local clone** using `git fsck --full`
4. **Create a bare repo** from the healthy clone
5. **Push to a new or reinitialized remote** using `git push --mirror`
6. **Verify the restored repo** with `git fsck` on a fresh clone

## Permanent Fix

1. **Set up repository mirroring:**
   ```bash
   # GitHub Enterprise mirror
   # Or set up a periodic mirror script
   git clone --mirror git@github.com:org/app.git /backups/app-mirror.git
   # Run daily via cron
   ```

2. **Implement git fsck monitoring:**
   ```bash
   # Add to cron job (daily)
   cd /repos/app && git fsck --full --no-dangling 2>&1 | \
     mail -s "Git Fsck Report - $(date)" admin@company.com
   ```

3. **Enable GitHub's repository protection features:**
   - Repository backup policies
   - Disable force push on protected branches

4. **Use git maintenance:**
   ```bash
   git maintenance start  # Enables background maintenance
   git maintenance run --task=full  # Manual full maintenance
   ```

5. **Implement geographic redundancy** with remote mirrors in different data centers.

## Monitoring

- Schedule daily `git fsck` on all critical repositories
- Monitor repository size growth (sudden changes may indicate issues)
- Alert on failed clone/push/pull operations
- Track git server disk health and I/O
- Monitor backup mirror sync status
- Alert if mirror lag exceeds threshold

## Security

- Ensure recovery process doesn't expose sensitive history
- Verify recovered repo has same access controls
- Check that no malicious objects were injected during corruption
- Audit who has access to local clones containing production code
- Revoke and rotate any credentials that may have been exposed

## Production Considerations

- **Recovery Time:** Target <1 hour from detection to full recovery
- **Data Loss:** Zero commit loss if healthy clone exists
- **CI/CD Impact:** All pipelines blocked until recovery - communicate ETA
- **Team Productivity:** Engineers cannot work until recovery completes
- **Compliance:** Maintain audit log of recovery actions
- **Cost of downtime:** Calculate engineering hours lost during outage

## Senior-Level Answer

"I'd first run `git fsck --full` on all available local clones to identify the healthiest one. Then I'd create a bare repository from that clone, verify its integrity, and push everything to a new or reinitialized remote using `git push --mirror`. For long-term prevention, I'd implement daily `git fsck` monitoring, set up repository mirroring for redundancy, and configure git maintenance tasks. The key insight is that git's distributed nature means corruption of the remote doesn't mean data loss - as long as at least one healthy clone exists."

## Architect-Level Answer

"This incident exposes a critical infrastructure gap. At the architectural level, we need: (1) Automated repository mirroring with integrity verification to multiple geographic locations, (2) Scheduled `git fsck` health checks integrated into monitoring with alerting, (3) Repository backup as part of disaster recovery with defined RPO/RTO, (4) Git server infrastructure hardening (RAID, ECC memory, proper shutdown procedures), (5) A recovery runbook that's tested quarterly. I'd also recommend evaluating GitHub's built-in backup features or third-party solutions like Veeam for GitHub that provide point-in-time recovery. The cost of prevention is orders of magnitude less than the cost of recovery."

## Follow-Up Questions

1. "What's the difference between a regular clone and a bare repository, and why do you use bare repos for recovery?"
2. "If all local clones were also corrupted, what would your recovery strategy be?"
3. "How would you recover specific branches or commits if only part of the repository is corrupted?"
4. "What is git's internal object model, and how does corruption at the object level manifest?"
5. "How would you design a zero-downtime migration of a Git repository to a new hosting platform?"
