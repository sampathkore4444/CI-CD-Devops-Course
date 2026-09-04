# 19 — Secret Management: HashiCorp Vault

> **Goal:** Master secret management — how banks protect credentials, API keys, and certificates.

---

## 🔍 Why Secret Management Matters

```
Without Secret Management:
  ❌ Passwords hardcoded in application.properties
  ❌ API keys in Git commits
  ❌ Certificates manually copied to servers
  ❌ No rotation, no audit trail
  ❌ One person knows all secrets

With Secret Management (Vault):
  ✅ Secrets stored in encrypted vault
  ✅ Dynamic secrets (short-lived credentials)
  ✅ Automatic rotation
  ✅ Complete audit trail
  ✅ Least-privilege access
  ✅ Zero standing privileges
```

---

## 🏗️ HashiCorp Vault Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    HASHICORP VAULT ARCHITECTURE                      │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    VAULT SERVER                               │  │
│  │                                                               │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │  │
│  │  │   Auth      │  │   Secret    │  │   Audit     │         │  │
│  │  │   Methods   │  │   Engines   │  │   Device    │         │  │
│  │  │             │  │             │  │             │         │  │
│  │  │ - LDAP      │  │ - KV        │  │ - File      │         │  │
│  │  │ - OIDC      │  │ - Database  │  │ - Syslog    │         │  │
│  │  │ - AppRole   │  │ - PKI       │  │ - Splunk    │         │  │
│  │  │ - K8s       │  │ - AWS/Azure │  │             │         │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘         │  │
│  │                                                               │  │
│  │  ┌─────────────────────────────────────────────────────┐    │  │
│  │  │                 POLICY ENGINE                        │    │  │
│  │  │                                                      │    │  │
│  │  │  path \"secret/data/payment/*\" {                      │    │  │
│  │  │    capabilities = [\"read\", \"list\"]                    │    │  │
  │  │  }                                                    │    │  │
│  │  └─────────────────────────────────────────────────────┘    │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                              │                                      │
│                              ▼                                      │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    ENCRYPTION BACKEND                         │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │  │
│  │  │ AWS KMS     │  │ Azure Key   │  │ Transit     │         │  │
│  │  │             │  │ Vault       │  │ Engine      │         │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘         │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🔑 Vault Secret Engines

### 1. KV Secret Engine (Static Secrets)
```bash
# Store a secret
$ vault kv put secret/payment/db-credentials \
    username=bank_admin \
    password=S3cur3P@ssw0rd!

# Retrieve a secret
$ vault kv get -field=password secret/payment/db-credentials
# S3cur3P@ssw0rd!

# List all secrets
$ vault kv list secret/payment/
# Keys
# ----
# db-credentials
# api-keys
# tls-certificates
```

### 2. Database Secret Engine (Dynamic Secrets)
```bash
# Configure database connection
$ vault write database/config/postgres \
    plugin_name=postgresql-database-plugin \
    connection_url=\"postgresql://{{username}}:{{password}}@prod-db:5432/banking\" \
    allowed_roles=payment-role \
    username=vault_admin \
    password=V@ultP@ss

# Create role with TTL
$ vault write database/roles/payment-role \
    db_name=postgres \
    default_ttl=1h \
    max_ttl=24h

# Generate dynamic credentials
$ vault read database/creds/payment-role
# Key                Value
# ---                -----
# lease_id           database/creds/payment-role/abc123
# lease_duration     1h
# lease_renewable    true
# password           A1b2-C3d4-E5f6-G7h8
# username           v-payment-role-abc123

# These credentials:
# - Are unique per request
# - Expire after 1 hour
# - Are automatically revoked
# - Are audit logged
```

### 3. PKI Secret Engine (Certificates)
```bash
# Configure PKI
$ vault secrets enable pki
$ vault write pki/config/urls \
    issuing_certificates=\"https://vault.bank.com/v1/pki/ca\" \
    crl_distribution_points=\"https://vault.bank.com/v1/pki/crl"

# Issue certificate
$ vault write pki/issue/bank-dot-com \
    common_name=\"payment.bank.com\" \
    ttl=720h

# Result:
# Key                Value
# ---                -----
# certificate        -----BEGIN CERTIFICATE-----
#                    MIIFjTCCBHWgAwIBAgIUK...
#                    -----END CERTIFICATE-----
# issuing_ca         -----BEGIN CERTIFICATE-----
#                    MIIFazCCBFOgAwIBAgIU...
#                    -----END CERTIFICATE-----
# private_key        -----BEGIN RSA PRIVATE KEY-----
#                    MIIEpAIBAAKCAQEA0Z3V...
#                    -----END RSA PRIVATE KEY-----
```

---

## 🏦 Banking End-to-End Examples

### E2E Example 1: Application Secret Injection

**Context:** Payment service needs database credentials from Vault.

```yaml
# ExternalSecret - sync Vault secrets to K8s
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: payment-db-secrets
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: payment-db-credentials
  data:
    - secretKey: db-username
      remoteRef:
        key: secret/payment/db-credentials
        property: username
    - secretKey: db-password
      remoteRef:
        key: secret/payment/db-credentials
        property: password
---
# Deployment using the secret
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
spec:
  template:
    spec:
      containers:
        - name: payment
          env:
            - name: DB_USERNAME
              valueFrom:
                secretKeyRef:
                  name: payment-db-credentials
                  key: db-username
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: payment-db-credentials
                  key: db-password
```

```bash
# Verify secret is synced
$ kubectl get secret payment-db-credentials -n production
# NAME                       TYPE     DATA   AGE
# payment-db-credentials     Opaque   2      5m

# Verify application can access
$ kubectl exec -it payment-abc123 -n production -- printenv DB_USERNAME
# bank_admin ✅

# Check Vault audit log
$ vault audit list
# path  TYPE  DESCRIPTION
# file/ file  File audit device

$ vault read sys/audit/file/config
# Audit log shows:
# 2026-09-04T10:00:00Z secret/read secret/payment/db-credentials - username=bank_admin
# 2026-09-04T11:00:00Z secret/read secret/payment/db-credentials - username=bank_admin
# 2026-09-04T12:00:00Z secret/read secret/payment/db-credentials - username=bank_admin
```

### E2E Example 2: Automatic Secret Rotation

**Context:** Rotate database password every 90 days per PCI-DSS.

```yaml
# CronJob for secret rotation
apiVersion: batch/v1
kind: CronJob
metadata:
  name: vault-secret-rotation
  namespace: vault
spec:
  schedule: "0 0 1 */3 *"  # Every 3 months
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: rotate
              image: vault:1.15
              command:
                - /bin/sh
                - -c
                - |
                  # Generate new password
                  NEW_PASSWORD=$(openssl rand -base64 32)
                  
                  # Update Vault
                  vault kv put secret/payment/db-credentials \
                      username=bank_admin \
                      password=$NEW_PASSWORD
                  
                  # Update database
                  vault write database/rotate-role/payment-role
                  
                  # Verify
                  echo "Secret rotated successfully"
              env:
                - name: VAULT_ADDR
                  value: "https://vault.bank.com"
          restartPolicy: OnFailure
```

### E2E Example 3: Vault Audit Trail for Compliance

**Context:** Regulators require proof that secrets are properly managed.

```bash
# Query Vault audit logs for last 30 days
$ vault audit list -detailed
# Vault audit device 'file/' is enabled at 'file:///var/log/vault/audit.log'

# Generate compliance report
$ python scripts/vault-compliance-report.py --period 30d

# ╔═══════════════════════════════════════════════════════════════════╗
# ║              VAULT COMPLIANCE REPORT (Last 30 Days)              ║
# ╠═══════════════════════════════════════════════════════════════════╣
# ║ Metric                              Value         Status          ║
# ╠═══════════════════════════════════════════════════════════════════╣
# ║ Total secret accesses               45,892        ✅ Normal       ║
# ║ Unique users                        23            ✅ Normal       ║
# ║ Failed authentication attempts      3             ⚠️ Review       ║
# ║ Dynamic secrets issued              1,247         ✅ Normal       ║
# ║ Dynamic secrets expired             1,247         ✅ All expired  ║
# ║ Certificates issued                 89            ✅ Normal       ║
# ║ Certificates expired                12            ✅ All rotated  ║
# ║ Secret rotation events              4             ✅ Scheduled    ║
# ║ Unauthorized access attempts        0             ✅ None         ║
# ║ Policy changes                      2             ✅ Reviewed     ║
# ╠═══════════════════════════════════════════════════════════════════╣
# ║ PCI-DSS Compliance: 100% ✅                                      ║
# ║ SOX Compliance: 100% ✅                                          ║
# ║ RBI Compliance: 100% ✅                                          ║
# ╚═══════════════════════════════════════════════════════════════════╝
```

---

## 📋 Interview Questions

### Q1: What is the difference between static and dynamic secrets?
**Answer:** **Static secrets** are fixed values stored in Vault (API keys, passwords). They persist until manually changed. **Dynamic secrets** are generated on-demand with a TTL (database credentials, AWS IAM keys). They're unique per request and automatically revoked. Banks prefer dynamic secrets because: (1) No shared credentials. (2) Automatic expiry. (3) Complete audit trail per request. (4) No credential leakage risk.

### Q2: How does Vault integrate with Kubernetes?
**Answer:** Three methods: (1) **Kubernetes Auth Method** — Vault verifies pod identity via Kubernetes API. (2) **External Secrets Operator** — syncs Vault secrets to K8s Secrets. (3) **CSI Driver** — mounts Vault secrets as volumes. Most banks use External Secrets Operator because: (1) Secrets auto-refresh. (2) Application code unchanged (reads K8s Secrets). (3) No sidecar needed.

### Q3: What is Vault's "transit" engine and when would you use it?
**Answer:** Transit engine provides **encryption as a service** — applications send data to Vault for encryption/decryption without managing keys. Use cases: (1) **PCI-DSS** — encrypt card numbers before storing in database. (2) **GDPR** — encrypt PII fields. (3) **Tokenization** — replace sensitive data with tokens. Benefits: application never sees encryption keys, key rotation is transparent.

### Q4: How do you handle Vault disaster recovery?
**Answer:** (1) **Vault Replication** — primary Vault replicates to DR cluster. (2) **Snapshot Backup** — `vault operator raft snapshot save` creates point-in-time backup. (3) **Auto-unseal** — KMS auto-unseal prevents seal stuck on DR failover. (4) **DR Failover** — promote DR cluster to primary. Banks typically run Vault in HA mode with 3+ nodes and cross-region replication.

### Q5: What is the principle of least privilege in Vault?
**Answer:** Every policy grants only the minimum access needed. Example: Payment service can only read `secret/payment/*`, not `secret/account/*`. Audit team can only list and read audit logs, not secrets. CI/CD pipeline can only read secrets for its specific service. This is enforced via Vault policies and RBAC. Regular policy reviews (quarterly) ensure no privilege creep.

---

## 📚 Summary

| Concept | Key Takeaway |
|---------|-------------|
| Vault | Centralized secret management |
| Static Secrets | Fixed values (API keys, passwords) |
| Dynamic Secrets | On-demand, auto-expiring (DB credentials) |
| PKI Engine | Automated certificate management |
| Audit Trail | Every access logged for compliance |
| Banking Relevance | PCI-DSS, credential rotation, least privilege |

**Next:** [20-Database-CICD.md](./20-Database-CICD.md) — Learn CI/CD for database changes.
