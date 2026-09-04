# 02 — Version Control & Git: The Foundation of CI/CD

> **Goal:** Understand why version control is the bedrock of CI/CD and master Git essentials.

---

## 🔍 What is Version Control?

**Version Control** is a system that records changes to files over time so you can recall specific versions later. Think of it as a **time machine for your code** — you can go back to any point in history.

### Why Version Control Matters for CI/CD

```
Without Version Control:
  Developer A: "I have the latest code"
  Developer B: "No, I have the latest code"
  Developer C: "Wait, who overwrote my changes?"
  → Chaos. No audit trail. No way to roll back.

With Version Control:
  Every change is recorded with who, when, and why
  Every change can be reviewed, approved, and reverted
  → Traceability. Collaboration. Safety.
```

---

## 🏗️ Git Architecture

```
┌─────────────────────────────────────────────────┐
│                 WORKING DIRECTORY                │
│            (Your actual files on disk)           │
└──────────────────────┬──────────────────────────┘
                       │ git add
                       ▼
┌─────────────────────────────────────────────────┐
│                  STAGING AREA                    │
│          (Files prepared for next commit)        │
└──────────────────────┬──────────────────────────┘
                       │ git commit
                       ▼
┌─────────────────────────────────────────────────┐
│              LOCAL REPOSITORY                    │
│         (History on your machine)                │
└──────────────────────┬──────────────────────────┘
                       │ git push
                       ▼
┌─────────────────────────────────────────────────┐
│            REMOTE REPOSITORY (GitHub/GitLab)     │
│       (Shared history everyone can access)       │
└─────────────────────────────────────────────────┘
```

---

## 🔑 Essential Git Commands

### 1. Setting Up
```bash
# Configure your identity (required for every commit)
git config --global user.name "Rajesh Kumar"
git config --global user.email "rajesh@bank.com"

# Initialize a new repository
git init
```

### 2. Basic Workflow
```bash
# Check what's changed
git status

# Stage a file
git add src/TransferService.java

# Stage all changes
git add .

# Commit with a meaningful message
git commit -m "feat: add NEFT transfer limit validation per RBI guidelines"

# Push to remote
git push origin main
```

### 3. Branching (The Power of CI/CD)
```bash
# Create a new branch
git checkout -b feature/instant-transfer

# Make changes and commit
git add .
git commit -m "feat: implement instant transfer feature"

# Push branch and create pull request
git push -u origin feature/instant-transfer

# Switch back to main
git checkout main

# Merge feature branch
git merge feature/instant-transfer

# Delete branch after merge
git branch -d feature/instant-transfer
```

### 4. Viewing History
```bash
# View commit history
git log --oneline --graph

# Example output:
# * a3b2c1d (HEAD -> main) feat: add NEFT limit
# * d4e5f6g fix: resolve login timeout
# * h7i8j9k feat: add 2FA authentication
# * l0m1n2o Initial commit
```

### 5. Undoing Changes
```bash
# Undo unstaged changes
git checkout -- filename.java

# Undo last commit but keep changes
git reset --soft HEAD~1

# Completely undo last commit and changes
git reset --hard HEAD~1

# Revert a commit (creates a new undo commit)
git revert abc123
```

---

## 🌿 Branching Strategies

### Strategy 1: GitFlow (Traditional, Good for Banks)
```
main (production)
  │
  ├── develop (integration branch)
  │     │
  │     ├── feature/transfer-limit
  │     ├── feature/fraud-detection
  │     └── feature/kyc-update
  │
  ├── release/v2.3.0
  │
  └── hotfix/critical-login-fix
```

**When to use:** Large teams, regulated environments, scheduled releases.

### Strategy 2: GitHub Flow (Simpler)
```
main (production)
  │
  ├── feature/add-transfer-limit
  ├── fix/login-timeout
  └── chore/update-dependencies
```

**When to use:** Smaller teams, frequent deployments, continuous deployment.

### Strategy 3: Trunk-Based Development (Advanced)
```
main (production)
  │
  ├── short-lived feature branches (< 2 days)
  │
  └── feature flags control visibility
```

**When to use:** Mature CI/CD, high-performing teams, continuous deployment.

---

## 🏦 Real-World Banking Scenarios

### Scenario 1: Hotfix for a Critical Payment Bug
**Context:** A bug in the UPI payment gateway causes 0.1% of transactions to fail silently. This was discovered by the monitoring team at 2:00 AM.

**Without Git (The Horror):**
```
Developer SSHs into production server
Edits the file directly
Server restarts
No one knows what changed
Next deployment overwrites the fix
Bug comes back
```

**With Git (The Right Way):**
```bash
# 1. Developer creates hotfix branch from main
git checkout main
git checkout -b hotfix/upi-transaction-fix

# 2. Fix the bug
# Edit: src/main/java/com/bank/upi/PaymentProcessor.java
git add src/main/java/com/bank/upi/PaymentProcessor.java
git commit -m "fix: prevent silent UPI transaction failures - BUG-4521"

# 3. Push and create PR
git push origin hotfix/upi-transaction-fix
# PR automatically triggers CI pipeline:
#   ✅ Unit tests pass
#   ✅ Integration tests pass
#   ✅ Security scan clean
#   ✅ Code review approved by senior dev

# 4. Merge to main → triggers CD pipeline → auto-deployed to production
# Total time: 45 minutes
```

**Example CI Output:**
```
Pipeline: hotfix/upi-transaction-fix
Stage 1: Build ✅ (12s)
Stage 2: Unit Tests ✅ 342/342 passed (28s)
Stage 3: Integration Tests ✅ 89/89 passed (1m 45s)
Stage 4: Security Scan ✅ 0 vulnerabilities (15s)
Stage 5: Deploy to Staging ✅ (45s)
Stage 6: Production Deploy ✅ Canary 5% → 25% → 100%
Monitoring: Error rate dropped from 0.1% to 0.001% ✅
```

### Scenario 2: Compliance Audit Trail
**Context:** RBI auditors request evidence that all production changes in the last 6 months were authorized, reviewed, and traceable.

**Without Git:**
- "We deployed from John's laptop"
- "The change was in an email attachment"
- "We're not sure who approved it"
- **Result:** Regulatory fine, remediation order

**With Git:**
```bash
# Complete audit trail for any change
$ git log --format="%H %an %ae %ad %s" --since="2026-03-01"

abc123 Rajesh Kumar rajesh@bank.com Mon Jun 1 10:30:00 2026 +0530 feat: add transaction limit validation
def456 Priya Sharma priya@bank.com  Tue Jun 2 14:15:00 2026 +0530 fix: resolve NEFT timeout issue
ghi789 Amit Patel amit@bank.com     Wed Jun 3 09:45:00 2026 +0530 chore: update security certificates

# Each commit links to:
# - Pull Request (who reviewed)
# - Ticket (business justification)
# - Pipeline log (test results)
# - Deployment record (when deployed)
```

**Audit Report Example:**
```
Change ID: PR-2847
Author: Rajesh Kumar (rajesh@bank.com)
Reviewer: Priya Sharma (priya@bank.com)
Business Justification: JIRA-BANK-4521 - RBI compliance
Commit: abc123def456
Pipeline: #1847 (all stages passed)
Deployed: 2026-06-01 11:15:00 IST
Environment: Production (cluster-prod-01)
Status: ✅ Approved | ✅ Tested | ✅ Deployed | ✅ Monitoring OK
```

### Scenario 3: Parallel Feature Development
**Context:** Three teams work simultaneously on: (1) Credit card rewards, (2) Loan calculator, (3) KYC update.

**The Branch Strategy:**
```
main
  ├── feature/credit-card-rewards (Team A)
  ├── feature/loan-calculator (Team B)
  └── feature/kyc-update (Team C)
```

**Workflow:**
```bash
# Team A: Credit Card Rewards
git checkout -b feature/credit-card-rewards
# ... 2 weeks of development ...
git push origin feature/credit-card-rewards
# Creates PR → CI runs → Review → Merge to main

# Team B: Loan Calculator (parallel, no conflicts)
git checkout -b feature/loan-calculator
# ... 3 weeks of development ...
git push origin feature/loan-calculator
# Creates PR → CI runs → Review → Merge to main

# Team C: KYC Update
git checkout -b feature/kyc-update
# ... 1 week of development ...
git push origin feature/kyc-update
# Creates PR → CI runs → Review → Merge to main

# Result: All 3 features developed independently
# Each merged separately with full test coverage
# No "merge hell" because branches were short-lived
```

---

## 🏦 Banking End-to-End Examples

### E2E Example 1: Hotfix for a Critical NEFT Transaction Bug

**Context:** A bug causes NEFT transactions to be recorded twice in the ledger. Must be fixed within 1 hour.

```bash
# 09:00 AM - Bug discovered by reconciliation team

# 09:05 AM - Developer creates hotfix branch
$ git checkout main
$ git pull origin main
$ git checkout -b hotfix/neft-double-entry

# 09:10 AM - Fix the bug
# File: src/main/java/com/bank/neft/TransactionService.java
# Bug: Race condition in double-entry posting
# Fix: Added synchronized block + database lock

$ git add src/main/java/com/bank/neft/TransactionService.java
$ git commit -m "fix: prevent double-entry in NEFT transactions\n\nRoot cause: Race condition in concurrent posting.\nAdded pessimistic lock on account_id during transaction.\nBug: BUG-7891\n\n🤖 Generated with Codebuff\nCo-Authored-By: Codebuff <noreply@codebuff.com>"

# 09:15 AM - Push and create PR
$ git push origin hotfix/neft-double-entry

# PR #2891 created automatically
# CI Pipeline triggers:
#   ✅ Build: 45s
#   ✅ Unit Tests: 247/247 passed
#   ✅ Integration Tests: 89/89 passed  
#   ✅ Security Scan: 0 vulnerabilities
#   ✅ Compliance Check: RBI limits validated

# 09:25 AM - Code review (expedited)
# Senior dev reviews in 10 minutes
# Approval from: Tech Lead + Compliance Officer

# 09:35 AM - Merge to main
$ git merge hotfix/neft-double-entry
$ git tag -a v2.3.1-hotfix -m "NEFT double-entry fix"
$ git push origin main --tags

# 09:40 AM - CD pipeline triggers
#   ✅ Docker build: bank/neft-service:v2.3.1-hotfix
#   ✅ Deploy to staging: verified
#   ✅ Deploy to production: canary 5% → 100%

# 09:55 AM - Monitoring confirms fix
# Error rate: 0.1% → 0.001%
# Transaction count: normalized
# Ledger: balanced

# 10:00 AM - Incident resolved
# Total time: 1 hour
# Customer impact: Minimal
# Audit trail: Complete
```

### E2E Example 2: Quarterly Regulatory Code Release

**Context:** RBI mandates quarterly compliance updates. Must track every change for audit.

```bash
# Quarterly Release Process (Git-centric)

# Week 1: Feature branches created
$ git checkout -b feature/q1-2026-rbi-compliance

# Features developed:
$ git log --oneline feature/q1-2026-rbi-compliance
a1b2c3d feat: add transaction limit validation
d4e5f6g feat: implement enhanced KYC verification
h7i8j9k feat: add suspicious activity reporting
l0m1n2o feat: update data retention policies
p3q4r5s feat: add audit trail encryption

# Week 2: Release branch created
$ git checkout -b release/q1-2026
$ git merge feature/q1-2026-rbi-compliance

# Week 3: Staging deployment & testing
$ git tag -a rc1 -m "Release candidate 1"
$ git push origin release/q1-2026 --tags
# CI: All 1,247 tests passed
# Compliance: RBI checklist validated
# Security: Penetration test clean

# Week 4: Production deployment
$ git tag -a v3.0.0 -m "Q1 2026 RBI Compliance Release"
$ git merge release/q1-2026 into main
$ git push origin main --tags

# Audit Report Generated:
$ git log --format='%H %an %ad %s' v2.9.0..v3.0.0

Commit: a1b2c3d | Author: Rajesh | Date: 2026-01-15 | feat: transaction limits
Commit: d4e5f6g | Author: Priya  | Date: 2026-01-18 | feat: KYC verification
Commit: h7i8j9k | Author: Amit   | Date: 2026-01-22 | feat: SAR reporting
Commit: l0m1n2o | Author: Sarah  | Date: 2026-01-25 | feat: data retention
Commit: p3q4r5s | Author: John   | Date: 2026-01-28 | feat: audit encryption

# Total: 5 features, 5 developers, 5 reviewers
# All changes traceable, auditable, compliant ✅
```

### E2E Example 3: Multi-Team Feature Integration

**Context:** 4 teams develop features simultaneously for a core banking upgrade.

```bash
# Team Structure:
# Team A: Account Management (10 devs)
# Team B: Payment Processing (8 devs)
# Team C: Loan Services (6 devs)
# Team D: Reporting & Analytics (5 devs)

# Branch Strategy (GitFlow):
main (production)
  ├── develop (integration)
  │     ├── feature/account-v2 (Team A)
  │     ├── feature/payment-v3 (Team B)
  │     ├── feature/loan-calc (Team C)
  │     └── feature/reporting (Team D)
  │
  ├── release/v4.0.0 (all teams merge here)
  └── hotfix/* (if needed)

# Week 1-3: Parallel development
# Team A works on account-v2 branch
$ git checkout -b feature/account-v2 develop
# 47 commits over 3 weeks
$ git push origin feature/account-v2
# PR #401-410 created and reviewed

# Team B works on payment-v3 branch
$ git checkout -b feature/payment-v3 develop
# 38 commits over 3 weeks
$ git push origin feature/payment-v3
# PR #411-418 created and reviewed

# Week 4: Integration
$ git checkout -b release/v4.0.0 develop

# Merge Team A
$ git merge feature/account-v2
# CI: 1,847 tests passed ✅

# Merge Team B
$ git merge feature/payment-v3
# CI: 2,103 tests passed ✅ (256 new tests)
# Conflicts resolved: 3 files (auto-resolved)

# Merge Team C & D similarly

# Week 5: Staging & Production
$ git tag -a v4.0.0 -m "Core Banking v4.0.0"
$ git checkout main
$ git merge release/v4.0.0
$ git push origin main --tags

# Result:
# - 4 teams, 29 developers
# - 160+ commits, all reviewed
# - 2,456 tests, all passing
# - 0 merge conflicts (short-lived branches)
# - Full audit trail for regulators
```

---

## 📋 Interview Questions

### Q1: What is the difference between `git merge` and `git rebase`?
**Answer:** `git merge` creates a new commit that combines two branches, preserving the full history of both. `git rebase` replays commits from one branch onto another, creating a linear history. In banking, `git merge` is preferred because it preserves the complete audit trail. `rebase` can rewrite history, which may violate compliance requirements.

**Example:**
```bash
# Merge (non-destructive, preserves history)
git checkout main
git merge feature/transfer-limit
# Result: merge commit with full history

# Rebase (linear, rewrites history)
git checkout feature/transfer-limit
git rebase main
# Result: linear history, but commits have new hashes
```

### Q2: How does Git handle merge conflicts?
**Answer:** When two branches modify the same lines of the same file, Git cannot automatically merge them — this is a **merge conflict**. Git marks the conflicting sections with `<<<<<<<`, `=======`, and `>>>>>>>` markers. The developer must manually resolve the conflict by choosing which version to keep.

**Example:**
```java
<<<<<<< HEAD (your branch)
public double calculateInterest(double amount, double rate) {
    return amount * rate * 365 / 360;  // 360-day year
=======
public double calculateInterest(double amount, double rate) {
    return amount * rate * 365 / 365;  // 365-day year
>>>>>>> feature/new-interest-calc
```

### Q3: What is a Pull Request (PR) and why is it critical in banking?
**Answer:** A Pull Request is a formal request to merge code from one branch into another. It triggers code review, automated testing, and approval workflows. 

In banking, PRs are critical because: 

(1) They enforce the **four-eyes principle** — every change must be reviewed by at least one other person. 
(2) They create an audit trail showing who approved what. 

(3) They link code changes to business requirements (JIRA tickets).

### Q4: What is `git stash` and when would you use it?
**Answer:** `git stash` temporarily saves uncommitted changes so you can work on something else without committing incomplete work.

**Example (Banking Context):**
```bash
# You're working on loan calculator
git stash
# Working directory is clean

# Urgent hotfix needed for payment gateway
git checkout -b hotfix/payment-fix
# ... fix and commit ...
git checkout main
git merge hotfix/payment-fix

# Return to your loan calculator work
git stash pop  # Your changes are back!
```

### Q5: How do you handle secrets (API keys, passwords) in Git?
**Answer:** **NEVER commit secrets to Git.** Use these approaches:
1. **Environment variables** — secrets are injected at runtime, not stored in code
2. **`.gitignore`** — exclude files like `application.properties` with credentials
3. **Secret management tools** — HashiCorp Vault, AWS Secrets Manager, Azure Key Vault
4. **Git hooks** — pre-commit hooks that scan for secrets before allowing commits

```bash
# .gitignore
*.properties
*.env
secrets/
credentials.json

# Pre-commit hook to detect secrets
#!/bin/bash
if git diff --cached --name-only | xargs grep -l "password\|api_key\|secret" 2>/dev/null; then
    echo "❌ Secrets detected in staged files! Use environment variables."
    exit 1
fi
```

---

## 📚 Summary

| Concept | Key Takeaway |
|---------|-------------|
| Git | Distributed version control, tracks every change |
| Branching | Parallel development without conflicts |
| PR/MR | Code review + audit trail + quality gate |
| Merge vs Rebase | Merge preserves history, rebase linearizes |
| Secrets | Never commit credentials; use env vars + vaults |
| Banking Relevance | Full traceability, compliance, four-eyes principle |

**Next:** [03-Continuous-Integration.md](./03-Continuous-Integration.md) — Learn how CI catches bugs early and keeps code healthy.
