# 33. Secret Leaked in CI/CD Pipeline Logs

## Scenario

At 2:15 PM, a developer pushes code to the `auth-service` repository. The CI pipeline (GitLab CI) runs and the build fails at the integration test stage. In the failure output, the full production PostgreSQL connection string — including the `postgresql://app:SuperSecretProdPassword@prod-db.internal:5432/maindb` — is printed in the pipeline log. The pipeline logs are accessible to 50+ developers across the organization. A developer in the same org (who left the company last week but may still have access) could see this. This password also happens to be reused for the production Redis cache and the payment gateway webhook secret. You literally just found out because a colleague joked "I see you're leaking passwords again" in the team Slack.

## Interviewer Question

"A production database password was leaked in your CI pipeline logs, visible to 50+ developers. What do you do RIGHT NOW, in what order, and how do you prevent this from ever happening again?"

## What I Should Think About

- This is a critical security incident (P0) with production impact
- The FIRST action is to check if the secret is still valid (test if it works)
- The SECOND action is to rotate the secret immediately — never assume "nobody saw it"
- Access logs visibility: anyone who had access to the pipeline logs saw it
- The password is reused across multiple systems — rotating means rotating all of them
- Root cause: how did the secret get into the code/logs? Probably an env var not set correctly, a `.env` file committed, or a print statement
- Standardize on a secrets management system (Vault, AWS Secrets Manager, etc.)
- Consider if the developer who left still has access — immediate access revocation
- Need to think about the blast radius: what data is in the databases?

## Ideal Answer

**Immediate Actions (First 15 minutes)**

1. **Verify the leak**: Read the pipeline logs — confirm what was leaked, when, and who has access
2. **Assess access scope**: Who has view access to these logs? (GitLab project members, CI visibility settings)
3. **Rotate the leaked credentials immediately** — this is non-negotiable, regardless of whether you think anyone saw it
4. **Revoke access of the departed employee** and any suspicious accounts
5. **Freeze the pipeline** — prevent new builds from running until secret handling is fixed
6. **Notify security team** — this is a reportable security incident

**Rotation Sequence (since password is reused):**

```bash
# 1. Rotate PostgreSQL password
aws rds modify-db-instance \
  --db-instance-identifier maindb \
  --master-user-password "<NEW-RANDOM-PASSWORD>" \
  --apply-immediately

# 2. Update the application's secrets
kubectl create secret generic app-db-secret \
  --from-literal=db_password="<NEW-RANDOM-PASSWORD>" \
  -n production --dry-run=client -o yaml | kubectl apply -f -

# 3. Restart pods to pick up new secret
kubectl rollout restart deployment/auth-service -n production

# 4. Rotate Redis password
kubectl exec redis-master-0 -n production -- \
  redis-cli CONFIG SET requirepass "<NEW-RANDOM-PASSWORD>"

# 5. Rotate payment gateway webhook secret
# (through payment provider's web console or API)
```

**Root Cause Analysis**

The leaked secret came from an application configuration error: the integration test logged the full connection string during a failure. The underlying issues:
- The .env file was accidentally committed to the repo (or env var not injected as expected)
- Failed assertion/exception logging printed the connection string
- The environment variable was passed as the wrong name, so the app fell back to a hardcoded/default value in a config file

**Prevention Strategy**

1. **Central secrets management**: Move all secrets to Vault/AWS Secrets Manager; never store in repo or CI env
2. **Secret scanning in CI**: Add gitleaks/trufflehog to scan commits before they hit the repo
3. **Log redaction**: Implement log redaction for credentials in all services
4. **Pipeline log access control**: Restrict who can view CI logs
5. **Secret rotation automation**: Automate monthly secret rotation

## Architecture

```
SECRET LEAK FLOW:
┌────────────┐   ┌────────────┐   ┌──────────────┐   ┌──────────────┐
│ Developer  │──→│ Git Push   │──→│   CI Logs    │──→│   50+ Devs   │
│ (in an org)│   │            │   │ Password in  │   │ can see the  │
└────────────┘   └────────────┘   │ failure msgs │   │ password     │
                                  └──────┬───────┘   └──────────────┘
                                         │
                                   ┌─────┴─────┐
                                   │ Production│
                                   │ PostgreSQL│ ← CREDENTIALS EXPOSED
                                   │ (payment  │
                                   │   data)   │
                                   └───────────┘

PREVENTION ARCHITECTURE:
┌──────────────┐  ┌──────────────────┐  ┌─────────────────────┐
│ Secret Vault │  │ Secret Scanner   │  │ Log Redaction Layer │
│ (AWS Secrets │  │ (gitleaks in CI) │  │ (regex for conn's)  │
│  Manager/    │  │ fails build on   │  │ masks credentials   │
│  Vault)      │  │ secret match     │  │ in all logs         │
└──────┬───────┘  └──────────────────┘  └─────────┬───────────┘
       │                                         │
       ├── DB passwords ─────────────────тельно─── │
       ├── API keys                              │
       ├── Webhook secrets                       │
       │                                         │
       └── Pods inject via Secrets →             │
           K8s + external-secrets-operator ──────┘
```

## Investigation

1. **Search all pipeline logs** for the leaked string (gitlab-rails console or log search tool, e.g., Loki/Grep on log storage)
2. **Check if the .env file is in version control**: `git log --all --oneline` checks; look for `.env`, `docker-compose.yml`, `application.properties` in the repo
3. **Review CI variable configuration**: Check GitLab CI/CD settings for variable visibility and masked variable settings
4. **Check who can access pipeline logs**: GitLab project member list, roles, visibility settings
5. **Search codebase for hardcoded credentials**: `rg "postgresql://"`, search for common secret patterns
6. **Check third-party tool integrations**: Did the log get forwarded to Slack, PagerDuty, or S3?
7. **Audit access logs**: Who accessed the pipeline log since the failure?
8. **Trace the leak origin**: Check if a CI environment variable was misconfigured, or if a test itself prints secrets

## Commands

```bash
# Search pipeline logs for leaked secret
# (via GitLab API)
curl -s --header "PRIVATE-TOKEN: <token>" \
  "https://gitlab.com/api/v4/projects/123/pipelines/456/jobs/789/trace" \
  | grep -i "postgresql://"

# Check for committed secrets in Git history
git log --all --oneline | while read line; do
  git show $line --name-only | grep -E "\.env|config.*secret|password"
done

# Scan repo with gitleaks before push (pre-commit hook)
gitleaks detect --source . --report-path gitleaks-report.json --log-opts="--all"

# Verify leaked secret is still valid (CAREFUL: only with security approval)
# NOTE: Do NOT test against production DB directly; use a read-only check
kubectl exec -it postgres-pod -n production -- \
  PGPASSWORD="<leaked-password>" psql -U app -d maindb -c "SELECT 1;"

# Rotate the password (PostgreSQL + K8s)
kubectl create secret generic app-db-secret \
  --from-literal=db_password="$(openssl rand -base64 32)" \
  --from-literal=redis_password="$(openssl rand -base64 32)" \
  -n production --dry-run=client -o yaml | kubectl apply -f -
kubectl rollout restart deployment/auth-service -n production

# Check log access control
curl -s --header "PRIVATE-TOKEN: <token>" \
  "https://gitlab.com/api/v4/projects/123/members/all" \
  | jq '.[] | {username: .username, access_level: .access_level}'

# Check CI variable masking
curl -s --header "PRIVATE-TOKEN: <token>" \
  "https://gitlab.com/api/v4/projects/123/variables" | jq .
```

## Root Cause

| Root Cause | How to Identify | Prevention |
|---|---|---|
| Secret in committed .env file | `.env`, `docker-compose.yml` in repo history | Add `.env*` to .gitignore; scan with gitleaks |
| Environment variable not masked in CI | CI/CD settings screen shows masked value off | Enable masking for CI variables |
| Failure logs print configuration values | Exception logger prints full connection string | Configure exception logging to redact credentials |
| Hardcoded default password in app code | Test passes because default works in dev | Remove fallback defaults; fail fast on missing env vars |
| Log aggregation tool stores secrets | Logs forwarded to Elasticsearch/Loki | Add redaction processor at ingestion point |
| Overly broad log access | 50+ developers can view pipeline logs | Restrict log access to production-critical roles |

## Immediate Mitigation

1. **Rotate all leaked secrets immediately** (DB, Redis, webhook) — do NOT wait for investigation
2. **Revoke departed employee's access** across all systems (GitLab, AWS, Slack, VPN)
3. **Revoke all other exposed credentials** — if it could have been exposed, rotate it
4. **Freeze pipeline builds** until secret handling is verified
5. **Identify affected customers/data**: Are there public-facing accounts whose data 
6. **Notify security, compliance, and legal** stakeholders

## Permanent Fix

1. **Adopt a secrets management tool** (HashiCorp Vault, AWS Secrets Manager, Doppler) — no secrets in repos or plaintext
2. **Add gitleaks to pre-commit hook and CI pipeline** — fail build on secret detection
3. **Mask all CI variables** in GitLab/GitHub/Azure DevOps
4. **Remove log statements that print configuration** — or add a redaction filter to log aggregator
5. **Restrict pipeline log access** — only pipeline owners and security team can view full logs
6. **Automate secret rotation** — quarterly rotation for production secrets
7. **Implement defense in depth** — network isolation plus credential rotation plus access control
8. **Store K8s secrets with external-secrets-operator** synced from Vault/Secrets Manager

## Monitoring

- **Alert on high count of `gitleaks found` findings** in CI pipeline
- **Alert on failed authentication attempts** against production DB (could indicate attacker using leaked password)
- **Monitor for unusual large data exports/dumps** from production database
- **Track secret rotation compliance** — alert if secrets exceed 90 days old
- **Monitor log aggregation for password patterns** — detect leaks in real-time
- **Audit pipeline log access** — track who downloads/viewed logs

## Security

- **PCI DSS**: Payment data exposure requires formal incident reporting
- **GDPR**: If customer PII leaked, may need to notify authorities within 72 hours
- **Least privilege**: 50 developers should not have access to production credentials
- **Password reuse policy**: Never reuse passwords between systems — this incident shows why
- **Third-party risk**: If the password leaked externally, consider monitoring for fraudulent activity
- **Air-gapped rotation**: Consider rotating periodically, not only after incidents

## Production Considerations

- **Reliability**: Rotation must be carefully staged to avoid availability impact
- **Cost**: Rotation automation (Vault/Secrets Manager) costs money but is essential
- **Operational**: Secure access management must be part of onboarding/offboarding process
- **Compliance**: Documentation of security incidents is required for audits
- **HA**: Secret rotation must be synchronized across nodes/sites — a secret changed on one node but not another causes outages

## Senior-Level Answer

"As a P0 security incident, I'd act in this order: (1) validate the leak by inspecting the pipeline log, (2) rotate the production DB password, Redis password, and payment webhook secret immediately — regardless of who may have seen them, (3) revoke access for users who recently departed, (4) freeze CI until the leak vector is identified, (5) find the root cause — typically a committed `.env` file or an unmasked CI variable. Then prevention: add gitleaks/trufflehog secret scanning to the pre-commit hook and CI pipeline, move all secrets to AWS Secrets Manager or Vault with external-secrets-operator on K8s, mask all CI variables, restrict pipeline log access, and configure log redaction so credentials never appear in output. Finally, automate quarterly rotation of all production secrets."

## Architect-Level Answer

"At the enterprise level, a leaked secret indicates three systemic failures: secret hygiene, logging hygiene, and access control. The architectural solution:

1. **Zero-secret architecture** — secrets never exist in repos, CI variables, or Kubernetes YAML. Instead, all secrets live in a central secrets platform (Vault with dynamic credentials — credentials generated on-demand, short-lived, and automatically revoked).
2. **Secret scanning as a quality gate** — every commit is scanned by gitleaks before merging; secrets in git history go through a purge process with `git filter-repo` + forced history rewrite.
3. **Log redaction architecture** — a centralized log redaction pipeline (e.g., OpenSearch with redaction processors, or a log-shipping agent like Fluent Bit with regex redaction) masks all credential patterns across the organization's log streams.
4. **Dynamic credentials** — the ultimate prevention is short-lived credentials that are worthless if leaked. Vault's database secrets engine generates time-limited DB passwords, so even a perfect leak has minimal blast radius.

Additionally, the enterprise should adopt a **Secret Management Standard** requiring: no secrets in code, dynamic credentials for databases, quarterly rotation, gitleaks in all pipelines, and incident runbooks for leaks."

## Follow-Up Questions

1. "How do you balance the need for secret scanning in CI with the performance cost it adds to the pipeline?"
2. "If the leaked password was in Git history (committed 2 years ago), how do you purge it from history and invalidate all copies?"
3. "How do you implement dynamic database credentials with HashiCorp Vault — what changes to the application architecture does this require?"
4. "Your compliance officer says the incident must be reported to the data protection authority within 72 hours. What information do you need to gather?"
5. "How do you test your secret rotation process without causing downtime? What's your rotation drill procedure?"