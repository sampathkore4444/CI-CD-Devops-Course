# 27. Large File Accidentally Committed - Repository Bloated

## Scenario

A developer on the data team accidentally committed a 2GB PostgreSQL database dump (`production_dump_2024_01_15.sql`) directly to the Git repository. The commit was pushed to `main` and is now in the history. The repository, which was previously 500MB, is now 3GB. The consequences are severe:

1. `git clone` now takes 45 minutes instead of 5 minutes
2. CI/CD pipelines are timing out during checkout (15-minute timeout)
3. Developers are running out of disk space on laptops
4. GitHub is showing repository size warnings
5. The `.git` directory on the build server is consuming 8GB of disk
6. The file was committed 3 weeks ago, and 200+ commits have been made since then

The developer who committed it has already been notified, but the damage is done. You need to remove the file from history without losing any subsequent commits, and prevent this from happening again.

## Interviewer Question

"A developer accidentally committed a 2GB database dump to Git. The repository is now bloated and CI/CD pipelines are timing out. How do you remove the file from history without losing any commits, and how do you prevent this from happening again?"

## What I Should Think About

- git filter-branch vs BFG Repo-Cleaner vs git filter-repo
- Impact on collaborators and open PRs
- Repository re-initialization after history rewrite
- Force push implications
- Alternative approaches (LFS, git-annex)
- Prevention strategies (.gitignore, pre-commit hooks)
- CI/CD pipeline adjustments
- Team communication during the rewrite

## Ideal Answer

**Phase 1: Choose the Right Tool**

`git filter-branch` is the original tool but is slow and dangerous for large repos. BFG Repo-Cleaner is faster and simpler. `git filter-repo` is the modern recommended replacement. For a 2GB file, BFG or git-filter-repo are the best choices.

**Phase 2: Clean the History**

Using BFG Repo-Cleaner (recommended for simplicity):

```bash
# Clone a fresh copy to work with
git clone --mirror git@github.com:org/repo.git repo-mirror.git
cd repo-mirror.git

# Use BFG to remove the large file
java -jar bfg.jar --strip-blobs-bigger-than 100M .

# Clean up and push
git reflog expire --expire=now --all
git gc --prune=now --aggressive
git push --force
```

**Phase 3: Handle Collaborators**

All collaborators must re-clone or reset their local copies. Force-pushing history rewrites means old local copies are incompatible.

**Phase 4: Prevent Recurrence**

Set up `.gitignore`, pre-commit hooks, and optionally Git LFS for large files.

## Architecture

```
Before Cleanup:
main ──●──●──●──[2GB dump]──●──●──●──●──● (200+ commits after)
                           ↑
                    The problem commit

After Cleanup:
main ──●──●──●──●──●──●──●──●──● (same commits, file removed from history)

Repository Size:
Before: 3GB (500MB code + 2GB dump in history)
After:  600MB (500MB code + overhead from packed objects)
```

## Investigation

1. Find the commit that introduced the large file
2. Check if the file is still referenced in any branches
3. Check if Git LFS is already configured
4. Check current .gitignore for existing rules
5. Assess how many collaborators will be affected
6. Check if any CI/CD artifacts reference the file
7. Verify no open PRs depend on the file

## Commands

```bash
# Step 1: Find the large file in history
git rev-list --objects --all | \
  git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' | \
  sed -n 's/^blob //p' | \
  sort -rnk2 | head -20
# Shows the largest objects in history

# Step 2: Find the specific commit
git log --all --pretty=format:'%h %s' --diff-filter=A -- "*production_dump*"
# Shows which commit added the file

# Step 3: Check if the file exists on any branches
git branch -a --contains <commit-hash>

# Step 4: Install BFG Repo-Cleaner
# Download from https://rtyley.github.io/bfg-repo-cleaner/
# Or via Homebrew:
brew install bfg

# Step 5: Create a mirror clone (safe to modify)
git clone --mirror git@github.com:org/repo.git repo-bfg.git
cd repo-bfg.git

# Step 6: Remove the large file using BFG
# Option A: Remove by filename
java -jar bfg.jar --delete-files "production_dump_2024_01_15.sql" .

# Option B: Remove by size (remove all files > 100MB)
java -jar bfg.jar --strip-blobs-bigger-than 100M .

# Step 7: Clean up the repository
git reflog expire --expire=now --all
git gc --prune=now --aggressive

# Step 8: Verify the file is gone
git rev-list --objects --all | \
  git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' | \
  sed -n 's/^blob //p' | \
  sort -rnk2 | head -5

# Step 9: Check new repo size
du -sh .
# Should be significantly smaller

# Step 10: Force push the cleaned history
git push --force --all
git push --force --tags

# Step 11: ALL COLLABORATORS must re-clone
echo "Repository has been cleaned. Please re-clone:"
echo "  rm -rf repo"
echo "  git clone git@github.com:org/repo.git"
echo "DO NOT use old local copies - they are incompatible."

# Step 12: Update .gitignore to prevent recurrence
echo "*.sql" >> .gitignore
echo "*.dump" >> .gitignore
echo "*.bak" >> .gitignore
echo "*.zip" >> .gitignore
echo "*.tar.gz" >> .gitignore
git add .gitignore
git commit -m "chore: add large file patterns to .gitignore"
git push origin main

# Step 13: Install pre-commit hook to prevent future large files
# Create .git/hooks/pre-commit (or use pre-commit framework)
cat > .git/hooks/pre-commit << 'EOF'
#!/bin/bash
MAX_SIZE=10485760  # 10MB in bytes
for file in $(git diff --cached --name-only); do
  if [ -f "$file" ]; then
    size=$(wc -c < "$file")
    if [ "$size" -gt "$MAX_SIZE" ]; then
      echo "ERROR: $file is $(($size/1048576))MB (max: 10MB)"
      echo "Use Git LFS for large files: git lfs track '$file'"
      exit 1
    fi
  fi
done
EOF
chmod +x .git/hooks/pre-commit
```

## Root Cause

1. **No .gitignore rules** for SQL dumps and large files
2. **No pre-commit hooks** to block large file commits
3. **Developer lack of training** on Git best practices
4. **No Git LFS** configured for binary/large files
5. **No CI/CD repository size checks**
6. **Large file culture** - team didn't establish conventions for large files

## Immediate Mitigation

1. **Clean the repository history** using BFG or git-filter-repo
2. **Force push** the cleaned history
3. **Notify all team members** to re-clone immediately
4. **Delete and recreate CI/CD caches** that reference the old repo
5. **Clear build server caches** of the old .git directory
6. **Monitor CI/CD pipelines** for successful checkout after cleanup

## Permanent Fix

1. **Set up .gitignore:**
   ```
   *.sql
   *.dump
   *.bak
   *.zip
   *.tar.gz
   *.gz
   *.mp4
   *.avi
   *.mov
   node_modules/
   __pycache__/
   *.pyc
   .env
   *.pem
   ```

2. **Install pre-commit framework:**
   ```yaml
   # .pre-commit-config.yaml
   repos:
     - repo: https://github.com/pre-commit/pre-commit-hooks
       rev: v4.4.0
       hooks:
         - id: check-added-large-files
           args: ['--maxkb=10240']
         - id: detect-private-key
         - id: detect-aws-credentials
   ```

3. **Configure Git LFS for necessary large files:**
   ```bash
   git lfs install
   git lfs track "*.psd"
   git lfs track "*.zip"
   git lfs track "*.sql"
   git lfs track "*.dump"
   git add .gitattributes
   git commit -m "chore: configure Git LFS for large files"
   ```

4. **Set up repository size monitoring:**
   ```bash
   # Add to CI/CD pipeline
   REPO_SIZE=$(du -sh .git | awk '{print $1}')
   echo "Repository size: $REPO_SIZE"
   # Alert if size exceeds threshold
   ```

5. **Establish team conventions:**
   - Large files go to object storage (S3, GCS)
   - Database dumps go to shared storage, not Git
   - Use Git LFS for binary assets that must be in the repo
   - Document the policy in CONTRIBUTING.md

## Monitoring

- Track repository size over time (alert on growth > 50%)
- Monitor CI/CD pipeline checkout times
- Track disk usage on build servers
- Alert on pre-commit hook violations
- Monitor Git LFS usage and storage costs

## Security

- Database dumps may contain sensitive data (PII, credentials)
- Verify the dump was removed from GitHub's cache/CDN
- Check if the dump was accessible to unauthorized users
- Rotate any credentials that may have been in the dump
- Audit access logs for the repository during the exposure period

## Production Considerations

- **Disk space:** Clean up build server caches immediately
- **CI/CD:** Pipelines may need manual cache invalidation
- **Collaboration:** All team members must re-clone (significant disruption)
- **Git LFS costs:** LFS storage has costs; evaluate if it's needed
- **Performance:** Repository clone time should return to ~5 minutes after cleanup
- **Compliance:** If the dump contained regulated data, this may need to be reported

## Senior-Level Answer

"I'd use BFG Repo-Cleaner to remove the large file from history. I'd clone a mirror, run BFG to strip the file, run aggressive garbage collection, and force push. All collaborators would need to re-clone. For prevention, I'd set up .gitignore patterns, install pre-commit hooks with size checks, and configure Git LFS for necessary large files. The key insight is that rewriting history requires coordination - everyone must re-clone, and old PRs may need to be recreated."

## Architect-Level Answer

"This incident reveals a gap in repository governance. The architectural solution involves multiple layers: (1) pre-commit hooks as the first line of defense, (2) CI/CD checks that fail builds with files over a size threshold, (3) Git LFS for legitimate large files with a clear policy on what qualifies, (4) repository size monitoring with alerting, (5) team training on Git best practices, and (6) an architecture decision record (ADR) documenting the large file policy. For the long term, I'd evaluate whether binary assets should be stored in object storage with Git tracking pointers, rather than in the repository itself. This is especially important as the team scales and repository performance becomes critical."

## Follow-Up Questions

1. "What's the difference between `git filter-branch`, BFG Repo-Cleaner, and `git filter-repo`?"
2. "How do you handle the situation where an open PR was based on a commit that included the large file?"
3. "Explain how Git LFS works internally and what happens when LFS storage is unavailable."
4. "How would you migrate an existing repository to use Git LFS without disrupting the team?"
5. "What's the impact of force-pushing a rewritten history on GitHub/GitLab pull requests and issues?"
