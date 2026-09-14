# 71. Secrets Exposed in Terraform State File

## Scenario

During a routine security audit, the InfoSec team discovered that the Terraform state file stored in S3 contains plaintext sensitive data — including RDS master passwords, API keys for third-party services, and TLS private keys. The state file is stored in a bucket accessible to 15 engineers across 3 teams. The state file has been in production for 18 months and has never been rotated. The security team has flagged this as a P2 vulnerability and wants it remediated within 2 weeks. Additionally, there are 12 environments (dev, staging, prod across 4 regions) each with their own state files.

## Interviewer Question

"Our Terraform state file in S3 contains plaintext secrets including database passwords and API keys. The state is accessible to the entire DevOps team. How do you migrate secrets out of Terraform state, implement proper secrets management, and prevent this from happening again?"

## What I Should Think About

- Current state: secrets in plaintext in state file, wide access to state
- State file cannot be easily edited in place — Terraform manages state
- Need to rotate all exposed secrets (they are compromised)
- Need to migrate to a secrets manager (AWS Secrets Manager, HashiCorp Vault)
- Need to implement `lifecycle ignore_changes` or remove secrets from Terraform entirely
- Need to lock down state file access (S3 bucket policy, state locking)
- Need to prevent future secret additions to state
- Consider using `terraform import` + `terraform state rm` for migration
- Audit trail: who accessed the state file and when
- Compliance implications (SOC2, PCI-DSS if banking)

## Ideal Answer

**Phase 1: Immediate Containment**
1. Restrict S3 bucket access immediately — only CI/CD service accounts
2. Enable S3 access logging and CloudTrail on the bucket
3. Rotate ALL exposed secrets immediately (DB passwords, API keys, TLS keys)
4. Check for any unauthorized access to the state file using CloudTrail

**Phase 2: Secrets Migration**
1. Move all secrets to AWS Secrets Manager (or HashiCorp Vault)
2. Update Terraform to reference secrets from the secrets manager using `data` sources
3. Remove secret values from Terraform configuration — use `aws_secretsmanager_secret_version` only for initial creation
4. Use `lifecycle { ignore_changes = [secret_string] }` to prevent Terraform from managing the secret value after creation
5. Alternatively, use `null_resource` with `local-exec` provisioner to set secrets only once

**Phase 3: State File Cleanup**
1. Use `terraform state rm` to remove resources that contain secrets
2. Re-import them without the secret values in Terraform config
3. Alternatively, create new state for secrets-related resources only
4. Delete old state file after confirming new state is correct

**Phase 4: Prevention**
1. Implement `tfsec` or `Checkov` in CI/CD pipeline to scan for secrets in Terraform
2. Add pre-commit hooks with `detect-secrets` or `git-secrets`
3. Enforce state file encryption with KMS key
4. Implement least-privilege IAM policies for state bucket access
5. Use workspace-level state isolation for sensitive environments

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    BEFORE (Insecure)                     │
│                                                          │
│  ┌──────────┐    plaintext secrets    ┌──────────────┐  │
│  │ Terraform │ ──────────────────────► │  S3 State     │  │
│  │ Config    │                         │  (15 users)   │  │
│  │ (secrets  │                         │  (plaintext)  │  │
│  │  in code) │                         └──────────────┘  │
│  └──────────┘                                            │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                    AFTER (Secure)                        │
│                                                          │
│  ┌──────────┐   data source   ┌───────────────────┐    │
│  │ Terraform │ ──────────────► │ AWS Secrets Manager│    │
│  │ Config    │                 │ (encrypted, audited│    │
│  │ (no       │                 │ , versioned)       │    │
│  │  secrets) │                 └───────────────────┘    │
│  └──────────┘                                            │
│       │                                                   │
│       │ state refs only                                   │
│       ▼                                                   │
│  ┌──────────────┐   encrypted    ┌──────────────┐       │
│  │ S3 State      │ ◄─────────── │ KMS Key       │       │
│  │ (CI/CD only)  │              │ (CMK)         │       │
│  │ (no secrets)  │              └──────────────┘       │
│  └──────────────┘                                       │
└─────────────────────────────────────────────────────────┘
```

## Investigation

1. **Identify all secrets in state file:**
   ```bash
   # Download state file
   aws s3 cp s3://terraform-state-bucket/prod/terraform.tfstate ./prod-state.json
   
   # Search for sensitive patterns
   grep -iE "(password|secret|api_key|private_key|token)" ./prod-state.json | head -50
   ```

2. **Check state file access logs:**
   ```bash
   # Check S3 access logs
   aws s3api get-bucket-logging --bucket terraform-state-bucket
   
   # Check CloudTrail for state file access
   aws cloudtrail lookup-events \
     --lookup-attributes AttributeKey=ResourceName,AttributeValue=prod/terraform.tfstate \
     --start-time $(date -d '30 days ago' +%Y-%m-%dT%H:%M:%SZ) \
     --max-results 100
   ```

3. **List all Terraform resources containing secrets:**
   ```bash
   # Pull current state and identify secret-bearing resources
   terraform state list | xargs -I {} terraform state show {} | grep -iE "(password|secret|key)"
   
   # Or more precisely
   terraform state pull | jq -r '.resources[] | select(.type == "aws_db_instance" or .type == "aws_secretsmanager_secret") | .instances[].attributes' | head -100
   ```

4. **Check for secrets in Terraform source code:**
   ```bash
   # Scan codebase
   grep -rn "password\|secret_key\|api_key\|private_key" *.tf --include="*.tf"
   
   # Use tfsec
   tfsec . --format json | jq '.results[] | select(.rule_id == "AVD-AWS-0098" or .rule_id == "AVD-AWS-0099")'
   ```

## Commands

```bash
# 1. Rotate RDS password
aws rds modify-db-instance \
  --db-instance-identifier prod-mysql \
  --master-user-password "$(aws secretsmanager get-random-password --password-length 32 --query 'RandomPassword' --output text)" \
  --apply-immediately

# 2. Create secrets in AWS Secrets Manager
aws secretsmanager create-secret \
  --name prod/database/master-password \
  --secret-string '{"username":"admin","password":"NEW_ROTATED_PASSWORD"}' \
  --kms-key-id alias/terraform-secrets

# 3. Remove secret resources from state (non-destructive)
terraform state rm aws_db_instance.main.master_password

# 4. Update Terraform config to use data source
cat > secrets.tf << 'EOF'
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "prod/database/master-password"
}

resource "aws_db_instance" "main" {
  # ... other config ...
  username = jsondecode(data.aws_secretsmanager_secret_version.db_password.secret_string)["username"]
  password = jsondecode(data.aws_secretsmanager_secret_version.db_password.secret_string)["password"]
  
  lifecycle {
    ignore_changes = [password]  # Don't manage password after initial set
  }
}
EOF

# 5. Add pre-commit hook
cat > .pre-commit-config.yaml << 'EOF'
repos:
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.83.5
    hooks:
      - id: terraform_tfsec
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets
        args: ['--baseline', '.secrets.baseline']
EOF

# 6. Restrict S3 bucket access
aws s3api put-bucket-policy --bucket terraform-state-bucket --policy '{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowCICDOnly",
    "Effect": "Deny",
    "Principal": "*",
    "Action": "s3:*",
    "Resource": ["arn:aws:s3:::terraform-state-bucket/*"],
    "Condition": {
      "StringNotLike": {
        "aws:PrincipalArn": "arn:aws:iam::*:role/terraform-execution-role"
      }
    }
  }]
}'
```

## Root Cause

| Root Cause | Elimination |
|---|---|
| Secrets hardcoded in Terraform variables/ configs | Use `variable "password" { sensitive = true }` and external secrets manager |
| No scanning for secrets in Terraform code | Add `tfsec`, `checkov`, `detect-secrets` to pre-commit and CI/CD |
| State file accessible to too many people | Restrict S3 bucket policy to CI/CD roles only |
| State file not encrypted at rest | Enable SSE-KMS encryption on S3 bucket |
| No secret rotation policy | Implement 90-day rotation for all secrets via Secrets Manager |
| No audit trail for state access | Enable CloudTrail + S3 access logging |

## Immediate Mitigation

1. **Restrict state file access** — update S3 bucket policy to deny all except CI/CD roles
2. **Rotate ALL exposed secrets** — treat them as compromised (DB passwords, API keys, TLS keys)
3. **Enable CloudTrail logging** on the state bucket
4. **Scan for unauthorized access** in the last 30 days via CloudTrail
5. **Notify security team** that rotation is complete

## Permanent Fix

1. Migrate all secrets to AWS Secrets Manager with KMS encryption
2. Update all Terraform modules to reference secrets via `data` sources
3. Add `lifecycle { ignore_changes }` to prevent Terraform from managing secret values
4. Implement pre-commit hooks and CI/CD scanning for secrets
5. Enforce least-privilege IAM policies for state bucket
6. Implement secret rotation schedules in Secrets Manager
7. Create a `SENSITIVE-README.md` documenting which resources contain secrets
8. Implement Terragrunt with encrypted backend configuration

## Monitoring

- **CloudTrail alerts** when state file is accessed outside CI/CD
- **S3 bucket access logging** with CloudWatch Logs
- **AWS Config rule** `encrypted-volumes` to ensure all EBS/RDS are encrypted
- **Secrets Manager** rotation status monitoring
- **tfsec/Checkov** scan results as CI/CD pipeline gates
- **Alert on** any `terraform state pull` or `terraform state mv` outside approved pipelines

## Security

- State file should never be accessible to humans directly — only through CI/CD
- All secrets should be encrypted at rest (KMS) and in transit (TLS 1.2+)
- Implement IAM policy conditions limiting state access to specific VPCs/IPs
- Use AWS CloudTrail Insights to detect unusual access patterns
- Consider HashiCorp Vault for dynamic secrets (short-lived DB credentials)
- Implement RBAC: different state files for different sensitivity levels
- Audit all Terraform plans before apply (mandatory PR review)
- Enable S3 Object Lock for state files (compliance requirement)

## Production Considerations

- **12 environments** means 12 state files — need automated secret rotation across all
- Coordinate secret rotation with application deployments to avoid downtime
- Use Terraform workspaces or Terragrunt for environment isolation
- Consider using `terragrunt run-all` for coordinated updates across environments
- Cost: Secrets Manager costs $0.40/secret/month + $0.05 per 10,000 API calls
- Document runbooks for emergency secret rotation
- Implement break-glass procedure for state file access during incidents
- Consider using AWS Systems Manager Parameter Store for non-sensitive config values

## Senior-Level Answer

"I'd implement a 3-phase remediation: immediate containment (restrict access, rotate all compromised secrets), migration (move secrets to AWS Secrets Manager, update Terraform to use data sources with lifecycle ignore), and prevention (pre-commit hooks, CI/CD scanning with tfsec/Checkov, least-privilege IAM). For ongoing governance, I'd implement automated secret rotation, state file access auditing via CloudTrail, and a secrets management policy. The key insight is that Terraform state should be treated as sensitive infrastructure — humans should never interact with it directly."

## Architect-Level Answer

"At an organizational level, I'd establish a Secrets Management Standard operating procedure that mandates: (1) No secrets in any IaC code or state files — all secrets go through a centralized secrets management platform, (2) Automated secret scanning at every stage — pre-commit, CI/CD pipeline, and post-deployment, (3) Ephemeral credentials where possible — use AWS IAM roles, OIDC, and dynamic secrets from Vault, (4) State file access architecture — move to a model where state is managed exclusively by CI/CD pipelines with no human access, using remote state data sources for cross-team references, (5) Compliance automation — AWS Config rules, SCPs to prevent S3 buckets without encryption, and regular penetration testing of IaC pipelines."

## Follow-Up Questions

1. "What's the difference between using AWS Secrets Manager vs HashiCorp Vault for this use case? When would you choose one over the other?"
2. "How do you handle secret rotation without causing downtime for applications that cache credentials?"
3. "A team member accidentally commits a secret to Git. What's your response process?"
4. "How do you manage secrets in a multi-cloud environment with both AWS and Azure?"
5. "What's your approach to dynamic secrets for database credentials in a Kubernetes environment?"
