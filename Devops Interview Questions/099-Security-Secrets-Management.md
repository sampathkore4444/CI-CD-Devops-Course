# 99. Secrets Management Across Environments

## Scenario

Your organization runs 5 environments (dev, test, staging, production, DR). Secrets are scattered across:
- Environment variables in CI/CD pipelines (GitHub Actions secrets, Jenkins)
- Kubernetes ConfigMaps
- Hardcoded values in application config files committed to the repo
- `.env` files on developers' machines
- Docker `.env` in images

A security audit flagged this as critical. The team needs to centralize secrets management. The company uses AWS as cloud provider with Kubernetes (EKS), and has HashiCorp Vault available in some teams. You must design a governance model, choose the tooling (Vault vs AWS Secrets Manager vs Sealed Secrets), and plan migration from scattered secrets.

## Interviewer Question

"Secrets are scattered across env vars, ConfigMaps, hardcoded code, and .env files across 5 environments. How do you centralize secrets management using Vault, AWS Secrets Manager, or Sealed Secrets?"

## What I Should Think About

- Evaluate the tool options: Vault, AWS Secrets Manager (SM), Kubernetes Sealed Secrets
- Governance: which secrets, which owners, which environments, rotation cadence, audit
- Migration plan from scattered → centralized
- Least privilege: app roles only access their own secrets
- Kubernetes integration: Vault Agent Injector, external-secrets operator, CSI Secrets Store
- CI/CD integration: secrets injected at runtime, never baked into images/env
- No secrets in git; no secrets in logs; no secrets in images
- DR: secrets must exist in DR region / recovery; cross-region replication
- Rotation policy and automation
- FinOps: cost of Secrets Manager vs self-hosted Vault
- Compliance: audit trail of access is required (PCI)

## Ideal Answer

**Step 1 — Governance first:**

1. Inventory all secrets: DB creds, API keys, TLS certs, tokens, SSH keys — for all 5 environments
2. Assign secret owners (team name) and classify by sensitivity (Public/Internal/Confidential/Restricted)
3. Define lifecycle: create → use → rotate → revoke; rotation cadence (90 days default); approval/audit policy

**Step 2 — Choose tooling:**

| Requirement | Vault | AWS Secrets Manager | Sealed Secrets |
|---|---|---|---|
| Multi-env / multi-cloud | yes | AWS-only | K8s-only |
| Dynamic/db creds | yes (dynamic) | no | no |
| Rotation automation | strong | strong (managed) | manual |
| K8s native | VIA CSI/injector | via external-secrets/CSI | yes (encrypt .template with k9s key) |
| On-prem / DR | yes | region-bound | yes (limpid) |
| Cost | self-hosted ops | ~$0.40/secret/mo | free |

Hybrid pragmatic path:
- **AWS Secrets Manager** for AWS-native + transparent rotation + KMS (best for DB creds)
- **Vault** if we need dynamic secrets, multi-cloud, or fine-grained policy + audit (choose for org-wide secrets)
- **Sealed Secrets** as the K8s-native fallback where we store only encrypted manifests in git (works even without a server; more a local-in-git pattern)

Decision: Centralize on **Vault** (governance + audit + multi-env), with **AWS Secrets Manager** as a backing store for AWS-specific infra (RDS creds rotation), and **Sealed Secrets** only for aced co-ex collaborations if the team already has the pattern.

**Step 3 — Migration:**

1. Build an inventory; map current secret usage
2. Create Vault paths per env (`secret/{env}/app/{app}/...`); onboard secrets with owner metadata
3. App integration: use Vault Agent Injector (inject secrets into pod env/files at runtime via K8s), or external-secrets operator
4. Enforce push gates: CI rejects builds/images that contain known secret patterns / stop baking `.env` into images
5. Cutover app by app, remove the old secret from the repo/env; verify at runtime
6. Full-cycle test: rotation works and breaks nothing

## Architecture

```
  CURRENT: scattered (BAD)
  ├── GitHub Actions → env var: DB_PASSWORD
  ├── Jenkins → env var
  ├── ConfigMap → value in plaintext
  ├── .env (dev machines / repo)
  └── hardcoded in app config

  TARGET: single control plane per decision
  ┌───────────────────────────────────────────────┐
  │                 VAULT                          │
  │   kv engine: secret/prod/app/api/credentials  │
  │   kv engine: secret/staging/...               │
  │   policy binding: app role ↔ path scope       │
  │   audit log (all reads logged)                │
  │   rotation: dynamic DB creds / PKI cert pool  │
  └───────────────────────┬───────────────────────┘
                          │ K8s integration:
          ┌───────────────▼────────────────┐
          │ external-secrets / CSI Driver  │
          │   - inject as mounted file     │
          │   - no env var, no ConfigMap   │
          └───────────────┬────────────────┘
                          │
        ┌─────────────┐   │   ┌────────────────┐
        │ EKS staging │   │   │ EKS production │
        └─────────────┘   │   └────────────────┘
                          │
        ┌─────────────────┴────────────────┐
        │ CI/CD injects at task-run time   │
        │   (vault CLI, short-lived tokens)│
        └──────────────────────────────────┘

  AWS-native fallback for RDS creds:
  RDS ↔ AWS Secrets Manager (managed rotation via Lambda)
  pod reads via external-secrets (same pod contract as Vault)
```

## Investigation

**Step 1: Inventory current secrets**
```bash
# find secrets in git history
gitleaks detect --source . --log-opts=--all -v
# public leak detectors: trufflehog, secret-scan, git-secrets

# find env vars / ConfigMaps in cluster
kubectl get secrets -A
kubectl get configmaps -A -o yaml | grep -iE "password|token|key"
# pipeline secrets (GitHub): gh secret list -R owner/repo
```

**Step 2: Classify and map usage**
```bash
# list which services use which secrets
grep -rE "DB_PASSWORD|API_KEY|ACCESS_TOKEN" --include="*.yaml" --include="*.env*" src config > inventory.txt
# flag which are: prod / non-prod, sensitive, shared, rotated?
```

**Step 3: Evaluate existing infra**
```bash
# is Vault already running anywhere?
kubectl get pods -A | grep vault
helm list -A | grep vault
# is external-secrets present?
kubectl get crd externalsecrets.external-secrets.io
# KMS key inventory
aws kms list-keys --region us-east-1
```

**Step 4: Decide integration per app**
```bash
# app runtime → Vault Agent sidecar / CSI
# app config → env via secrets
# which apps are Spring/Node/etc. — check for existing enc-support
```

## Commands

```bash
# Vault: enable kv v2 engine
vault secrets enable -path=secret kv-v2

# Create structured secrets per env
vault kv put secret/prod/api/db \
  username=app_user \
  password="$(openssl rand -base64 32)"

# Policy: minimal access per app
vault policy write api-app - << 'EOF'
path "secret/data/prod/api/*" {
  capabilities = ["read", "list"]
}
path "secret/data/staging/api/*" {
  capabilities = ["read"]
}
EOF

# App role for K8s auth
vault auth enable kubernetes
vault write auth/kubernetes/config \
  kubernetes_host=https://kubernetes.default.svc \
  token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)"

# External-Secrets Operator: create ExternalSecret CR
kubectl apply -f - << 'EOF'
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: api-db-secret
  namespace: api
spec:
  refreshInterval: 1h
  secretStoreRef: {name: vault-store, kind: SecretStore}
  target: {name: api-db-secret}
  data:
    - secretKey: DB_USERNAME
      remoteRef: {key: prod/api/db, property: username}
    - secretKey: DB_PASSWORD
      remoteRef: {key: prod/api/db, property: password}
EOF

# AWS Secrets Manager (for RDS rotation)
aws secretsmanager create-secret \
  --name prod/api/db --secret-string '{"username":"app","password":"xyz"}'
aws secretsmanager rotate-secret \
  --secret-id prod/api/db --rotation-rules '{"AutomaticallyAfterDays":30}'

# Sealed Secrets (K8s-native fallback)
kubeseal --format yaml < secret.yaml > sealed-secret.yaml
kubectl apply -f sealed-secret.yaml
```

## Immediate Mitigation

```bash
# 1. Lock down exposures NOW:
#    - remove secrets from git history (filter-repo) + rotate what leaked
#    - delete plaintext ConfigMaps / env vars; replace with mount-based secrets
# 2. Fast path: put current secrets into AWS Secrets Manager (migrate manually)
aws secretsmanager put-secret-value --secret-id app/db --secret-string '{"password":"..."}'
#    - wire apps via external-secrets (agent-less) quickly
# 3. Gate CI: remove secret env vars from pipelines; inject short-lived tokens
# 4. Verify apps still connect; mark cabinet not yet rotated
```

## Permanent Fix

1. **Choose primary tool** (Vault recommended; AWS SM for AWS-native DBs; Sealed Secrets only for k8s-manifest-only)
2. **SecretStore per env** wired via external-secrets/CSI
3. **No secrets in git** — enforce via push protection; purge and rotate anything leaked
4. **Rotation automation**: Vault dynamic creds / managed rotation; scheduled rotation for static (quarterly)
5. **Audit**: every secret read logged (Vault audit, CloudTrail for SM); compliance reports
6. **Least privilege**: per-env, per-app paths; no shared super-secrets
7. **DR**: secrets replicated to DR region / restore-test; add a DR secret vault + failover access
8. **Developer workflow**: `vault CLI` helper; never `.env` in repos; local dev uses local Vault dev or env-injected creds

## Monitoring

```bash
# Metrics/events to monitor:
# - Secret access audit trails (Vault audit log, CloudTrail for SM)
# - Rotation status (per secret: last rotation date, success/failure)
# - Secrets in git: gitleaks CI + GitHub secret scanning alerts
# - Unused/abandoned secrets flagged
# - DR secret vault replication lag
# - Error on secret fetch (app can't read secret) — quick fail

# Alerts:
# - Rotation FAILED → page (it breaks apps if missed)
# - High-frequency reads of a specific secret → anomaly review
# - Secret in git leak detected → 噂 page
# - Vault/SecretsManager unreachable → availability alert
```

## Security

- Encryption at rest: Vault uses shamir + KMS auto-unseal; SM uses KMS keys
- Encryption in transit: TLS everywhere (Vault), SecretsManager HTTPS
- Least privilege: app roles scoped; revocation on separation
- Rotation is the correct response to any leak — not deletion
- Avoid baking secrets into images; use read-only mounts + short-lived access
- Audit: who read prod DB creds? (Vault audit → SIEM correlation)
- Compliance: PCI/SOC2 demand documented encryption, rotation, access reviews — this design satisfies them

## Production Considerations

- **HA**: Vault is HA via Consul (or Vault Enterprise integrated storage); SM is regional → plan cross-region secret replication
- **Scalability**: Vault handles thousands of apps; SM scales fine (cost per secret)
- **Cost**: self-hosted Vault = ops cost; SM ≈ $0.40/secret/month; Sealed secrets ~free but weak rotation
- **Compliance**: audit logs retained; periodic access review; rotation evidence
- **Operational**: rotate before every major incident/leak; a secrets-drive simulated drill (rotate prod DB creds with zero downtime)
- **DR**: secrets replicated / recreatable — DR vault policy, unseal keys, historical secrets (required by financial DR tests)

## Senior-Level Answer

"First, I'd inventory everything — git history, ConfigMaps, env vars, CI secrets — using gitleaks + a grep audit, then lock down leaks immediately by rotating and removing. Then I'd centralize on Vault (unless AWS-native SM is preferred for AWS-managed DBs), with external-secrets/CSI as the K8s integration so pods read secrets as mounted files, never env vars. Governance: secret-per-team-per-env path model with least-privilege policies, rotation automation (dynamic DB creds in Vault or managed rotates), and audit logging wired to SIEM. Migration moves app-by-app with runtime verification. For DR I'd ensure secrets are re-creatable or replicated, and I'd enforce the gold rule: nothing sensitive lives in git, images, or ConfigMaps — ever."

## Architect-Level Answer

"Secrets management is a governance problem, not just a tool decision. I'd establish a secret taxonomy (owner, env, sensitivity, rotation) plus a lifecycle contract: create → grant → use → rotate → revoke — all audited. Design-wise, I'd use Vault as the org-wide control plane (dynamic secrets, policy engine, audit) with AWS Secrets Manager where managed rotation is the best fit (RDS), and external-secrets as the integration layer so all apps share one consumption contract. Kubernetes get only short-lived, permission-scoped access via Vault agent/CSI; CI/CD uses short-lived tokens, not stored creds. DR means replicating secret store + validating unseal/recovery. Add FinOps guardrails (secret count, cost) and a rotation drill. The goal: any secret leak is cheap to contain because everything is short-lived, auditable, and centralized."

## Follow-Up Questions

1. "How do dynamic Vault DB credentials differ from static Secrets Manager rotation for less-secure workloads that hold a connection open for hours?"
2. "You have 300 apps to migrate — how do you prioritize and automate the relocation without a big-bang?"
3. "How does Vault's audit log satisfy PCI evidence for 'who read financial secrets'?"
4. "What happens to secrets when a developer leaves? Map revocation across your design."
5. "How do you keep a 5-environment model DR-consistent when only prod and DR exist in a regional failure?"