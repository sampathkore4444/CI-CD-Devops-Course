# 95. Production Database Password Committed to GitHub

## Scenario

A developer accidentally committed a production database password to a public GitHub repository 3 hours ago. The commit contains `.env` with `PROD_DB_PASSWORD` and AWS credentials (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`). 50 people have access to the repo. The repo is public. The password protects a PostgreSQL database with user PII (transactions). Bot scrapers have 3 hours of head start to scan for exposed credentials. The commit is still public. You're the DevOps/SRE lead.

## Interviewer Question

"A developer accidentally committed a production database password to a public GitHub repository 3 hours ago. 50 people have access to the repo. How do you handle the complete security incident response?"

## What I Should Think About

- Incident response stages: Contain → Analyze → Eradicate → Recover → Post-mortem
- CONSTANT: treat every secret as COMPROMISED. Rotation is mandatory, not optional
- Order of operations matters — some secrets guard infrastructure; passwd rotation first? Or access?
- The repo is public → immediate removal is necessary but NOT sufficient. History/GitHub caching
- GitHub secret scanning / push protection — can't rely on it, but reference
- Credential rotation for the DB immediately — but ALSO the AWS keys that grant S3/EC2 access
- Validate no breach via logs/Audit (CloudTrail, DB connection logs, query logs)
- 50 people = internal leak risk too
- Legal/compliance: disclosure obligations, PII exposure timelines
- Key rotation in dependent systems (app config, secrets manager) and coordinated cutover

## Ideal Answer

**Phase 1 — Contain (immediately, in parallel):**
1. **Public → private NOW** (or temporarily take down the repo). Stop the bleeding to the wide internet.
2. **Rotate the database password FIRST** and immediately. Any password that has been public is compromised regardless of bot detection. Update all apps to the new secret via secrets manager.
3. **Rotate the AWS access keys** — deactivate old keys, create new ones, update them in affected systems.
4. **Revoke exposed tokens** if any (GitHub PATs, npm tokens etc., if they were in the file).

**Phase 2 — Analyze:**
1. Check GitHub audit logs — who accessed the repo, forks created, etc.
2. Check DB audit/connection logs for suspicious connections from unknown IPs
3. Check AWS CloudTrail for unauthorized API activity using those keys (before rotation)
4. Confirm whether the secret was used by scanning reputation/leak-check services (e.g., TruffleHog results, HaveIBeenPwned-style sources) to bound blast radius
5. Check internal systems for reuse of the same password (people may reuse)

**Phase 3 — Eradicate:**
1. `git filter-repo` / history rewrite on the repo (or rebase/squash) to purge secret from history
2. Force-push cleaned history; contact GitHub support if forks exist / repo is public to invalidate cached copies
3. Delete all exposure branches/tags, and remove from GitHub cached diffs

**Phase 4 — Recover:**
1. Ensure all services use rotated secrets
2. Add secrets scanning (Secret Scanning / gitleaks) as CI gate
3. Re-verify applications connect with new credentials; run smoke tests

**Phase 5 — Post-mortem & prevention:**
1. Why was the secret in the repo? .gitignore miss? Secret still in env file?
2. Implement secrets manager everywhere; enforce push protection; add pre-commit hooks; use `.env.*` in .gitignore
3. Notify compliance/PII stakeholders per policy

## Architecture

```
  INCIDENT TIMELINE (3h window):
  ┌─────────────────────────────────────────────────────────────┐
  │ T-3h    dev runs git push, public repo, .env commited      │
  │         secrets: PROD_DB_PASSWORD + AWS keys               │
  │ T-2.5h  GitHub mirrors secret; scanners crawl              │
  │ T-1h    someone notices commit / secret scanning alert     │
  │ T+0     SRE on call, THIS IS THE incident                  │
  └─────────────────────────────────────────────────────────────┘

  CONTAINMENT ORDER (critical):
  ┌───────────────┐   ┌──────────────────┐   ┌───────────────┐
  │ repo public→  │   │ ROTATE DB        │   │ ROTATE AWS    │
  │ private        │   │ password (NOW)   │   │ keys (NOW)    │
  └───────────────┘   └──────────────────┘   └───────────────┘
        │                      │                      │
        ▼                      ▼                      ▼
  ┌─────────────────────────────────────────────────────────────┐
  │ Update all dependent apps via Secrets Manager / Vault        │
  │  - app1, app2 consume secret from manager                    │
  │  - coordinated no-downtime rotation (dual-scheme)            │
  └─────────────────────────────────────────────────────────────┘

  ROTATION PATTERN (safe rotation):
  APP → SecretsManager → current secret
  rotate: SecretsManager writes NEW secret →
  app polls/reloads → old invalidated.

  AWS key rotation:
  key1 (ACTIVE) → key2 (ACTIVE) → deactivate key1 (wait 24h) → delete
```

## Investigation

**Step 1: Repo — make it private & confirm access scope**
```bash
gh repo edit OWNER/REPO --visibility private
# or: Settings → Change visibility → Private

# Check forks / downloads / clones:
gh api repos/OWNER/REPO/forks --paginate
```

**Step 2: Identify the exposed secrets**
```bash
# Search the repo for all secrets
gh secret list -R OWNER/REPO
git log --all -p -- .env | grep -E "PASSWORD|AWS_|TOKEN|SECRET"
# Use gitleaks to enumerate exposed keys from history
gitleaks detect --redact=false --log-opts=--all
```

**Step 3: DB — check and rotate. First check connection logs**
```bash
# Check RDS/Postgres connection logson all servers
# RDS:
aws rds describe-db-log-files --db-instance-identifier prod-db
aws rds download-db-log-file-portion --db-instance-identifier prod-db \
  --log-file-name error/postgresql.log > /tmp/pg.log
grep -i "connection\|authentication" /tmp/pg.log | tail -100

# Identify unknown IPs / connections around T-2h (post-exposure)
grep -i "postgresql" /tmp/pg.log | awk '{print $3}' | sort | uniq -c
```

**Step 4: AWS — check CloudTrail for the leaked key**
```bash
# Find access key usage after exposure window
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=AccessKeyId,AttributeValue=<LEAKED_KEY> \
  --start-time $(date -d '-4 hours' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date +%Y-%m-%dT%H:%M:%SZ) --output table

# Check for EC2 launches, S3 reads, IAM changes
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName \
  AttributeValue=RunInstances --query 'Events[?UserIdentity.accessKeyId==`...`]'
```

**Step 5: Check 50 collaborators for possible compromise**
```bash
gh api repos/OWNER/REPO/collaborators --paginate
gh api repos/OWNER/REPO/teams
# Confirm no one added/removed keys/SSH without approval
```

## Commands

```bash
# STEP A: Make repo private + freezes pushes
gh repo edit OWNER/REPO --visibility private --enable-pushes=false

# STEP B: Rotate DB password via RDS (managed)
aws rds describe-db-instances --db-instance-identifier prod-db
aws rds modify-db-instance --db-instance-identifier prod-db \
  --master-user-password 'NEW-strong-random-password' --apply-immediately

# STEP C: Rotate AWS access key (two-key rotation)
aws iam create-access-key --user-name svc-prod-api \
  --output json | tee key2.json
# update apps with key2; wait >24h
aws iam update-access-key --access-key-id <LEAKED_KEY> --status Inactive
# after validation:
aws iam delete-access-key --user-name svc-prod-api --access-key-id <LEAKED_KEY>

# STEP D: Purge from git history (rewrite)
git filter-repo --invert-paths --path .env
# or migrate: git clone, filter-branch then force push + squash history
git push --force --all

# GitHub support to invalidate cached diffs (if public w/ forks)
# https://support.github.com/contact

# STEP E: Deploy new secrets to apps
aws secretsmanager rotate-secret --secret-id prod/db/credentials
# apps pull new secret; DB updated; old invalid.

# STEP F: Remediate scanning exposure (prevent recurrence)
# gitleaks in CI:
gitleaks detect --source . --report-path gitleaks-report.json --redact
# GitHub secret scanning + push protection
# blockfile-patterns via hooks: .pre-commit-config? or Codeowners.
```

## Root Cause

Primary: **Secrets in source** — dev had `.env` with prod credentials not in `.gitignore`, committed publicly.
Contributing:
- No pre-commit secret scanning / push protection
- `.env` not committed to ignore-file (or committed before adding ignore)
- No secrets manager for prod credentials at creation time
- Public-by-default repos; no repo scanning for secrets

## Immediate Mitigation

```bash
# 1. Make repo private (or take down)
gh repo edit OWNER/REPO --visibility private

# 2. ROTATE immediately:
#    - DB password (RDS modify; then update all services)
#    - AWS access keys (create new, deactivate old)
#    - Any tokens in the file (GitHub PATs, Slack tokens, etc.)
#      → revoke on their dashboards

# 3. Kill the old credentials everywhere:
aws secretsmanager update-secret --secret-id prod/db \
  --secret-string '{"username":"app_user","password":"<new>"}'
# app reconnects with new secret (no app restart needed with reload)

# 4. Verify no unauthorized access:
#    - DB connection logs: check unexpected IPs post-exposure
#    - CloudTrail: check key usage
#    - If found: security + compliance team escalation

# 5. Keep incident channel open, provide status to stakeholders.
```

## Permanent Fix

1. **Secrets manager / Vault for ALL production credentials** — never in env files or code
2. **Enforce secret scanning** (GitHub secret scanning + push protection; gitleaks pre-commit/CI)
3. **GitGuardian-style monitoring** for exposed secrets in GitHub
4. **`.gitignore` codified** & a pre-commit hook rejecting `.env*` at commit time
5. **Repo visibility policy** — default private; only public when approved
6. **Credential rotation policy** — regular rotation cadence (e.g., DB password every 90 days / on any exposure)
7. **Least-privilege IAM** for app roles (short-lived credentials)
8. **Awareness training** and post-mortem with the team; run a "secret spill" drill

## Monitoring

```bash
# Continuous secret detection:
# - GitHub secret scanning alerts
# - gitleaks CI gate
# - GitGuardian / TruffleHog in CI

# Auth/log monitoring:
# - CloudTrail + Athena alerting: unusual key usage, new EC2 launches
# - RDS log aggregation: failed authentication attempts, unknown source IPs
# - VPC flow log analysis for anomalous outbound connections

# Alerts:
# - Secret scanning hit in repo → P1 page
# - New access key created → audit notification
# - Failed DB auths > threshold → warning

# Dashboards: secret scanner findings, repo visibility change events,
# privilege escalation events, public repo list
```

## Security

- Blast radius: assume bots grabbed it in minute one. Rotation is the only defense
- Do NOT merely delete the commit; history rewrite + GitHub cache purge where forks exist
- Check sub/cibbrary reuse of the leaked password across services
- Consider PII disclosure: notify DPO/compliance/legal, and data subjects if required by GDPR/CCPA — document decision
- Enable GitHub 2FA / SSO for all collaborators; revoke keys of accounts that interacted
- Preserve artifacts for forensic analysis but secure them (SIEM integration)

## Production Considerations

- **HA**: rotation must be non-disruptive — use dual-scheme (parallel old+new accepted briefly) so apps don't blip
- **Reliability**: secrets rotation should be automated; manual rotation invites drift
- **Cost**: AWS Key Management System / Secrets Manager costs minor vs incident impact
- **Compliance**: incident response coordination, documentation, and disclosure obligations — time-boxed
- **Operational**: maintain a runbook for secret leaks with clear cross-team escalation (#security, DBAs, app owners)
- **Governance**: define who owns each secret; rotation ownership is a dated policy

## Senior-Level Answer

"The secret is burned — no amount of deleting the commit undoes the 3-hour exposure. Rotation is non-negotiable: I'd make the repo private, rotate the DB password (via RDS), rotate the AWS keys using the two-key method, and revoke any tokens in the file — simultaneously, with dual-scheme grace so services don't blip. Then I'd scan CloudTrail and DB logs for unauthorized access to size the blast radius, rewrite git history with `git filter-repo`, and purge GitHub's cached diffs. Long-term, the fix is systemic: secrets manager, push protection, pre-commit secret scanning, private-by-default repos, and mandated credential rotation. 'The commit got removed' is not the endpoint — rotation plus proof that nothing was accessed is the actual closure."

## Architect-Level Answer

"This is an incident-response + security-model problem. Architecturally, the fix is to make secrets non-bearer: lease-based credentials (Vault dynamic secrets / short-lived AWS tokens) so that a leak has a bounded shelf life. That converts an incident into a low-severity alert. I'd add secret scanning as a CI/CD gate (GitGuardian/gitleaks) plus GitHub push protection, and enforce private-by-default repos with a public-repo approval. The incident response itself needs a cross-functional runbook with an incident commander, documented time-boxed rotation, audit of access (CloudTrail + DB logs + collaborator review), history rewrite including fork/cache invalidation, and a post-mortem feeding the security backlog. Data governance also matters: if PII or card data was at risk, involve compliance for disclosure timelines. The strategic goal is to design so that any leaked credential is cheap to invalidate and automatically rotated."

## Follow-Up Questions

1. "If someone had already exfiltrated data using the leaked password, what forensic steps do you take for legal/compliance?"
2. "How do you rotate a password safely when NO downtime is allowed and multiple apps might hold stale copies?"
3. "What's the difference between making a repo private and securely deleting a commit from vs rewriting history, and when does GitHub even store failed pushes?"
4. "How does GitHub's secret scanning + push protection interact with forks, and where does it fall short (if a contributor uses a fork) ?"
5. "Design the automated rotation for the DB password using AWS Secrets Manager including how apps pick up the new value without restart."