# 62. S3 Bucket Access Denied - IAM Issue

## Scenario

At 2 PM, the payment processing application running on AWS ECS (Fargate) suddenly starts logging `AccessDenied` errors when trying to read files from an S3 bucket (`payment-receipts-prod`). The ECS tasks use an IAM role (`ecs-payment-task-role`) that has an S3 access policy. Nobody changed the IAM policies, the S3 bucket policy, or the ECS task definition. Other microservices can read their S3 buckets fine. The app team is now on a conference call saying "it was working yesterday — nothing changed." Your job is to identify why S3 access is denied and restore service quickly, then figure out what changed.

## Interviewer Question

"An application on ECS suddenly cannot read from its S3 bucket. Nothing was intentionally changed. Other apps access their buckets fine. Walk me through how you'd troubleshoot S3 access denied issues — including IAM, bucket policies, encryption, SCPs, and the S3 query paths."

## What I Should Think About

- S3 access denied follows evaluation logic: explicit deny beats allow, SCP beats everything
- Multiple layers: IAM identity policy, resource (bucket) policy, SCP (Organizations), session policies, encryption keys (KMS), bucket ownership (ACLs)
- Permission boundaries can silently block even when the main policy looks right
- KMS key access is separate — S3 can be allowed but the KMS key denies `kms:Decrypt`
- Public access block settings can change behavior
- Bucket policy can have a `Principal` that excludes the role
- The S3 bucket might have a bucket policy with an explicit `Deny` (e.g., deny if not using HTTPS)
- Cross-account access issues: bucket in account A, app in account B — both bucket policy AND identity policy needed
- Use `aws s3 ls` with `--debug` or the dry-run with `--dryrun` to see denial specifics on the CLI
- Check CloudTrail S3 data events

## Ideal Answer

**Step 1 — Reproduce and get the exact error**

```bash
# Try to access the bucket with the same role
aws s3 ls s3://payment-receipts-prod/ --no-sign-request 2>&1   # use actual creds!
aws s3 ls s3://payment-receipts-prod/ --dryrun
aws s3api get-object --bucket payment-receipts-prod --key receipts/2024/01/file.pdf /tmp/file.pdf

# With verbose logging (shows the exact AccessControlList / reason)
set -x; aws s3 ls s3://payment-receipts-prod/; set +x

# Simulate policy to find what's failing (IAM policy simulator)
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::<acct>:role/ecs-payment-task-role \
  --action-names s3:GetObject \
  --resource-arns arn:aws:s3:::payment-receipts-prod/*
```

**Step 2 — Check IAM role policy**

```bash
# Get the role's attached policies
aws iam list-attached-role-policies --role-name ecs-payment-task-role
aws iam get-policy-version \
  --policy-arn arn:aws:iam::<acct>:policy/PaymentS3Access \
  --version-id v17

# Inline policies
aws iam list-role-policies --role-name ecs-payment-task-role

# Permission boundary check!
aws iam list-role-policies --role-name ecs-payment-task-role
aws iam get-role --role-name ecs-payment-task-role --query 'Role.PermissionsBoundary'
```

**Step 3 — Check bucket policy**

```bash
aws s3api get-bucket-policy --bucket payment-receipts-prod
aws s3api get-bucket-acl --bucket payment-receipts-prod
aws s3api get-public-access-block --bucket payment-receipts-prod
aws s3api get-bucket-encryption --bucket payment-receipts-prod
```

**Step 4 — Check KMS key permissions (a top cause of "S3 Access Denied")**

```bash
# Is the bucket encrypted with SSE-KMS? If so, the role needs kms:Decrypt on the key
aws s3api get-bucket-encryption --bucket payment-receipts-prod

# Check key policy grants
aws kms get-key-policy --key-id <key-id> --policy-name default
aws kms describe-key --key-id <key-id>
```

**Step 5 — Check SCP (if using AWS Organizations)**

```bash
# As org admin
aws organizations list-policies-for-target --target-id <ou-id> --filter SERVICE_CONTROL_POLICY
```

**Step 6 — Check CloudTrail for the denial event**

```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=GetObject \
  --start-time "$(date -u -d '24 hours ago' +%Y-%m-%dT%H:%M:%SZ)"
```

Look at `errorCode: AccessDenied`, then check whether it's an identity policy, resource policy, or key policy denying.

## Architecture

```
    S3 Access — What can DENY (evaluation order)
    ────────────────────────────────────────────

    Request: ECS task → s3:GetObject on payment-receipts-prod

    ┌───────────────────┐  ┌──────────────────────┐
    │ 1. SCP (Org)       │  │ 2. Permission Boundary│
    │ explicit deny wins │  │ explicit deny wins   │
    └─────────┬─────────┘  └──────────┬───────────┘
              └───────────┬───────────┘
                          ▼
    ┌─────────────────────────────────────┐
    │ 3. IAM identity policies (role)     │
    │   allow/deny for the principal      │
    └──────────────────┬──────────────────┘
                       │
    ┌─────────────────┐│┌──────────────────┐
    │ 4. Bucket policy ││ 5. Session policy │
    │  resource-level  ││  (STS assume)     │
    └────────┬────────┘└──────┬────────────┘
             │                │
             └───────┬────────┘
                     ▼
    ┌─────────────────────────────────────┐
    │ 6. KMS key policy (if SSE-KMS)      │
    │   kms:Decrypt must be allowed       │
    └──────────────────┬──────────────────┘
                       ▼
    ┌─────────────────────────────────────┐
    │ Explicit deny at ANY layer → DENY   │
    │ Allow requires ALL applicable layers│
    └─────────────────────────────────────┘
```

## Investigation

1. **Reproduce** the failure exactly (same role, same bucket, same key) and capture the exact error message
2. **IAM Policy Simulator** — the fastest way to find which policy is denying access
3. **Check the IAM role's policies** — attached, inline, permission boundary
4. **Check bucket policy** for explicit denies and principal matching
5. **Check default encryption** — if SSE-KMS, verify the role can decrypt with the KMS key
6. **Check AWS Organizations SCPs** — account could have been moved to a more restrictive OU
7. **Check service control / session policies** — ECS task roles with a session policy via `sts:AssumeRole` tags
8. **Check if the bucket was recreated/replaced** (new bucket ARN? different region? unversioned?)
9. **Check if account was cross-account** — was the role granted access to a different account's bucket, and the trust changed?
10. **Check CloudTrail** for the `AccessDenied` error with full details

## Commands

```bash
# 1. Duplicate the error
aws s3api head-object --bucket payment-receipts-prod --key receipts/2024/01/file.pdf

# 2. IAM Policy Simulator via CLI
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::<acct>:role/ecs-payment-task-role \
  --action-names s3:GetObject s3:ListBucket \
  --resource-arns arn:aws:s3:::payment-receipts-prod arn:aws:s3:::payment-receipts-prod/* \
  --output json | jq '.EvaluationResults[] | {EvalActionName, EvalDecision, OrganizationsDecisionDetail}'

# 3. See the role and its policies
aws iam get-role --role-name ecs-payment-task-role --output json
aws iam list-attached-role-policies --role-name ecs-payment-task-role
aws iam list-role-policies --role-name ecs-payment-task-role

# 4. Bucket policy + encryption
aws s3api get-bucket-policy --bucket payment-receipts-prod
aws s3api get-bucket-encryption --bucket payment-receipts-prod
aws s3api get-public-access-block --bucket payment-receipts-prod

# 5. Check KMS key access
aws kms get-key-policy --key-id <key-id> --policy-name default --output text

# 6. CloudTrail — find the denial
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceName,AttributeValue=payment-receipts-prod \
  --lookup-attributes AttributeKey=EventName,AttributeValue=GetObject \
  --start-time "$(date -u -d '3 hours ago' +%Y-%m-%dT%H:%M:%SZ)"

# 7. Confirm which identity the ECS task is actually using
aws sts get-caller-identity
```

## Root Cause (most likely culprits)

| Root Cause | How to Identify | Fix |
|---|---|---|
| KMS key permissions changed/rotated | Bucket encrypted SSE-KMS, role has s3 but lacks kms:Decrypt | Add `kms:Decrypt` to role or fix key policy |
| Permission boundary added | `get-role` shows PermissionsBoundary | Adjust boundary to allow S3 |
| Bucket policy has explicit Deny (e.g., https-only, public blob block) | Bucket policy with `Effect: Deny` | Adjust policy |
| SCP denies S3 | Organizations policy updated; simulate shows OrganizationsDecision=Deny | Fix SCP at OU/account level |
| Role's trust policy changed so TaskRoleArn different | ECS service uses different task role than before | Update ECS service task role / trust |
| Bucket deleted & recreated with different ownership | Bucket ARN changed; account ID in policy no longer matches | Update policies to match new ARN |
| Bucket is now cross-account and role has no cross-account grant | require both identity policy AND bucket policy principal | Add bucket policy principal allowing role |
| Public Access Block changed | Response shows access blocked by S3 | If intended private, keep blocked |
| Condition in policy (e.g., `aws:SourceAccount`, `aws:SourceArn`) | Condition in effect denies non-matching principal | Align condition values |

## Immediate Mitigation

The fastest restoration path — do these in order:

1. **If it's a KMS key issue**, grant the role a temporary allow (least scope) on the key:
   ```bash
   aws kms put-key-policy --key-id <key-id> --policy-name default --policy file://fixed-key-policy.json
   # Add: {"Effect":"Allow","Principal":{"AWS":"arn:aws:iam::<acct>:role/ecs-payment-task-role"},"Action":"kms:Decrypt","Resource":"*"}
   ```

2. **If it's a bucket policy deny**, you can't override a deny — fix the policy directly:
   ```bash
   # Get current, edit, re-put
   aws s3api get-bucket-policy --bucket payment-receipts-prod > /tmp/bucket-policy.json
   # Edit /tmp/bucket-policy.json to correct the deny
   aws s3api put-bucket-policy --bucket payment-receipts-prod --policy file:///tmp/bucket-policy.json
   ```

3. **If the deny is at SCP level** it may block even your admin — likely you can't override from the member account; escalate to org admin.

4. **While fixing, reduce application impact** — if the S3 data is static, use a failover read volume or repo mirrored in ECR, or temporarily switch app config to a working bucket/prefix until the fix lands.

## Permanent Fix

1. **Move S3 access into an IaC-managed, least-privilege pattern** — policies stored in Terraform, CI applies them, drift detection on bucket policies
2. **Use paths/prefix-based IAM policies** (not `s3:* /*`):
   ```json
   {"Effect":"Allow","Action":["s3:GetObject"],"Resource":["arn:aws:s3:::payment-receipts-prod/*"]}
   ```
3. **Split responsibilities**: AWS S3 bucket owner manages bucket policy; app teams manage IAM — document the two-way responsibility
4. **Add tests**: quickly verify with `aws iam simulate-principal-policy` in CI before deploying IAM changes
5. **Encryption plan**: if using SSE-KMS, document/review KMS key policies and IAM roles together as one unit
6. **Alarm on AccessDenied error rate** in S3 data plane
7. **Runbook**: S3 access denied triage doc for on-call with the exact commands above

## Monitoring

```yaml
# CloudWatch for S3 data-plane events
- alert: S3AccessDeniedSpike
  expr: sum(logevents("S3DataEvents", '"AccessDenied"')) > 10 in 5m
  # CloudTrail data events → CloudWatch Logs stream
  severity: warning

- alert: KMSDecryptDenials
  expr: kms_errors(kms:Decrypt, AccessDenied) > 0
  for: 5m
  severity: critical
```

Enable CloudTrail **data events** for the bucket (`s3:GetObject`, `s3:PutObject`) so every denial is recorded and alertable.

## Security

- Least privilege on identity and bucket policies is the baseline
- Set **S3 Block Public Access** at account level
- Use **condition keys**: `aws:SourceVpce`, `aws:SourceAccount`, `aws:SecureTransport`
- Enable **default encryption** (SSE-KMS or SSE-S3)
- Use **Object Ownership/BucketOwnerEnforced** to prevent ACL-based confusion
- Rotate IAM user keys vs. using roles — prefer roles for ECS/EKS
- Separate buckets per environment/data classification
- **Glacier access**: monitor for unusual `RestoreObject` — watch for rg ransomware-style delete/overwrite of objects (enable S3 Object Lock for compliance)

## Production Considerations

- **Cost**: Data transfer and GET pricing — use CloudFront or S3 Transfer Acceleration where needed
- **Reliability**: Versioning enabled + lifecycle policies; test restore from Glacier (can take hours)
- **HA**: Replicate cross-region if DR required (`s3crr`), use dual-stack endpoints in multi-VPC
- **Operational**: central S3 access policy store; approve changes in code review
- **Compliance**: Data events in CloudTrail for audit; S3 Object Lock for immutable records
- **Cross-account**: use **S3 Access Points** to simplify per-tenant policies (VPC origin)
- **Encryption**: never operate S3 with SSE-KMS without auditing key policy grant changes

## Senior-Level Answer

"When an app suddenly loses S3 access with no intentional changes, I look for the layers in S3's effective permission model: IAM identity policy, bucket policy, permission boundaries, SCPs, session policies, and finally KMS key policy if SSE-KMS is enabled. The fastest diagnostic is the IAM Policy Simulator — it tells you which layer denied the request. I'd also check the KMS key immediately; 'S3 AccessDenied' is very often actually a `kms:Decrypt` denial. Then check CloudTrail data events to confirm the exact resource being denied. The fix is usually to restore the missing allow at the correct layer: grant kms:Decrypt, fix the bucket policy principal, or adjust a permission boundary. The mitigation is to restore access fast; the permanent fix is managing all these policies through IaC with drift detection so nothing changes silently."

## Architect-Level Answer

"S3 access-denied failures are a systems design problem, not a single-role problem. The architecture should make permission mutations visible and verifiable: (1) all IAM, bucket policies, KMS key policies, and SCPs defined in Terraform and reviewed in code; (2) CI pipeline runs `access-analyzer` findings + policy simulator against live roles after any IAM change; (3) CloudTrail data events for critical buckets logged and alerted; (4) a documented cross-account trust matrix for owned buckets. I'd standardize on least-privilege prefix-based IAM grants, enable BPA, and force SSE-KMS with central key tags. Finally, runbooks like this one give on-call a repeatable path. The strategic point: treat S3 security as an integrated permission system (identity + resource + keys + organizations) and test it continuously, not as a single policy that you fix in isolation."

## Follow-Up Questions

1. "What is the exact evaluation logic when an IAM identity policy allows an action but a bucket policy denies the same action?"
2. "If the bucket is encrypted with SSE-KMS, what are the minimum permissions the ECS task role needs for a read-only operation?"
3. "How does an SCP interact with an identity-based allow, and can you override an SCP deny?"
4. "Explain permission boundaries and session policies, and how they affect `aws s3 ls` behavior."
5. "A developer says the bucket is public but your app can't access it. How do you use S3 Access Analyzer and CloudTrail to determine the real issue?"