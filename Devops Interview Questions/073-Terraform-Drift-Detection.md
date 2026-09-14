# 73. Infrastructure Drift Detection and Remediation

## Scenario

During a P1 incident at 3 AM, an on-call engineer manually modified security groups directly in the AWS console to restore service. Three days later, during a routine `terraform plan`, you discover 47 resources with drift. Some changes were legitimate emergency fixes (security group rules), others were accidental (EC2 instance type changed, an S3 bucket was deleted). The drift spans 3 environments and involves 15 different resource types. You need to evaluate each change, decide whether to accept or revert, and implement drift detection to prevent future occurrences.

## Interviewer Question

"We found 47 drifted resources after manual console changes. How do you detect, evaluate, categorize, and remediate infrastructure drift while preventing it from happening again?"

## What I Should Think About

- Drift detection methods (terraform plan, AWS Config, custom scripts)
- Categorizing drift: intentional vs. accidental vs. malicious
- Accepting drift vs. reverting to IaC
- Terraform state manipulation (state rm, import, taint)
- Drift prevention (console access restriction, approval workflows)
- Multi-environment drift management
- Change management process integration

## Ideal Answer

**1. Detection**
- Run `terraform plan` across all workspaces/environments
- Use AWS Config to continuously detect drift
- Schedule weekly drift detection scans
- Integrate drift detection into CI/CD pipeline

**2. Evaluation**
- Categorize each drift as: intentional, accidental, or unknown
- For intentional changes: update Terraform code to match reality
- For accidental changes: revert to Terraform state
- For unknown: investigate before acting

**3. Remediation**
- Use `terraform refresh` to sync state with reality (for intentional changes)
- Use `terraform import` to bring new resources under management
- Use `terraform state rm` to remove orphaned resources
- For accidental changes: `terraform apply` to revert

**4. Prevention**
- Restrict console access (IAM policies, SCPs)
- Implement drift detection alerts
- Require all changes through Terraform/PR review
- Use AWS Service Control Policies (SCPs) to prevent manual changes

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│               DRIFT DETECTION PIPELINE                   │
│                                                          │
│  ┌──────────┐    ┌──────────┐    ┌──────────────────┐  │
│  │ Schedule  │───►│ Terraform │───►│ Drift Detection  │  │
│  │ (Cron)    │    │ Plan      │    │ Report           │  │
│  └──────────┘    └──────────┘    └────────┬─────────┘  │
│                                           │              │
│                    ┌──────────────────────┐│              │
│                    ▼                      ▼▼              │
│  ┌─────────────────────┐  ┌──────────────────────────┐  │
│  │ AWS Config          │  │ Slack/PagerDuty Alert     │  │
│  │ (Continuous)        │  │ (Human Review Required)   │  │
│  └─────────────────────┘  └──────────────────────────┘  │
│                                                          │
│  ┌─────────────────────────────────────────────────┐    │
│  │              REMEDIATION WORKFLOW                 │    │
│  │                                                   │    │
│  │  Drift Detected → Categorize → Approve → Apply   │    │
│  │      │              │          │         │        │    │
│  │      ▼              ▼          ▼         ▼        │    │
│  │  Intentional    Accept &    PR Review  terraform  │    │
│  │  Accidental     Update Code  Required   apply     │    │
│  │  Unknown        Revert                              │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

## Investigation

1. **Run terraform plan across all workspaces:**
   ```bash
   # For each environment
   for env in dev staging prod; do
     cd environments/$env
     terraform init -upgrade
     terraform plan -detailed-exitcode -out=drift-$env.tfplan 2>&1 | tee drift-$env.log
     echo "Exit code: ${PIPESTATUS[0]}"
   done
   ```

2. **Categorize drifted resources:**
   ```bash
   # Extract drift details
   terraform show -json drift-prod.tfplan | jq -r '.resource_changes[] | select(.change.actions[] != "no-op") | {address: .address, actions: .change.actions, type: .type}' | head -50
   
   # Count by action type
   terraform show -json drift-prod.tfplan | jq -r '.resource_changes[] | select(.change.actions[] != "no-op") | .change.actions[]' | sort | uniq -c
   ```

3. **Cross-reference with CloudTrail for change history:**
   ```bash
   # Find who made changes
   aws cloudtrail lookup-events \
     --lookup-attributes AttributeKey=EventName,AttributeValue=AuthorizeSecurityGroupIngress \
     --start-time "2024-01-15T00:00:00Z" \
     --max-results 50 | jq '.Events[] | {time: .EventTime, user: .Username, resources: .Resources}'
   ```

4. **Generate drift report:**
   ```bash
   # Create categorized report
   terraform show -json drift-prod.tfplan | jq -r '
     .resource_changes[] | 
     select(.change.actions[] != "no-op") |
     "Resource: \(.address)\nType: \(.type)\nActions: \(.change.actions | join(", "))\n"
   ' > drift-report.txt
   ```

## Commands

```bash
# 1. Refresh state to match reality (no apply)
terraform refresh

# 2. Accept intentional drift - update code to match
# Edit Terraform code to reflect actual state
terraform plan  # Verify only intended changes

# 3. Revert accidental changes
terraform apply -target=aws_security_group.old_config

# 4. Import resources not in state
terraform import aws_s3_bucket.new_bucket my-new-bucket

# 5. Remove orphaned resources from state
terraform state rm aws_instance.terminated_server

# 6. Taint resource for recreation
terraform taint aws_instance.corrupted_server

# 7. Use move to rename resources
terraform state mv aws_instance.old_name aws_instance.new_name

# 8. Set drift detection alert (AWS Config rule)
aws configservice put-config-rule --config-rule '{
  "ConfigRuleName": "terraform-drift-detection",
  "Source": {
    "Owner": "AWS",
    "SourceIdentifier": "CLOUDTRAIL_ENABLED"
  },
  "Scope": {
    "ComplianceResourceTypes": ["AWS::EC2::SecurityGroup"]
  }
}'

# 9. Disable console changes via SCP
aws organizations create-policy --content '{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyConsoleChanges",
    "Effect": "Deny",
    "Action": [
      "ec2:AuthorizeSecurityGroupIngress",
      "ec2:RevokeSecurityGroupIngress",
      "ec2:ModifyInstanceAttribute"
    ],
    "Resource": "*",
    "Condition": {
      "StringNotEquals": {
        "aws:PrincipalArn": "arn:aws:iam::*:role/terraform-execution"
      }
    }
  }]
}' --type SERVICE_CONTROL_POLICY --name "DenyManualInfraChanges"
```

## Root Cause

| Root Cause | Elimination |
|---|---|
| Console access too broad | Restrict via IAM + SCP |
| No drift detection process | Implement weekly automated scans |
| Emergency changes bypass IaC | Create break-glass procedure with post-incident IaC update |
| No change approval process | Require PR review for all infrastructure changes |
| State file out of sync | Run `terraform refresh` regularly |
| No documentation of manual changes | Require post-incident documentation |

## Immediate Mitigation

1. **Categorize all 47 drifted resources** — intentional vs. accidental vs. unknown
2. **Revert accidental changes** immediately via `terraform apply`
3. **Accept intentional changes** — update Terraform code to match reality
4. **Document all changes** in incident postmortem
5. **Restrict console access** for emergency changes to break-glass role only

## Permanent Fix

1. Implement weekly automated drift detection scans
2. Use AWS Config rules for continuous drift monitoring
3. Restrict console changes via IAM policies and SCPs
4. Require all changes through Terraform with PR review
5. Create break-glass procedure for emergencies (with mandatory post-incident IaC update)
6. Implement drift alerts via Slack/PagerDuty

## Monitoring

- **AWS Config** — continuous drift detection for critical resources
- **Terraform Cloud** — built-in drift detection (if using Terraform Cloud)
- **Custom Lambda** — weekly `terraform plan` with Slack notification
- **CloudTrail** — alert on manual infrastructure changes
- **Prometheus metrics** — `terraform_drift_resources_total` gauge

## Security

- All manual changes should be logged and audited
- Break-glass access should require MFA and be time-limited
- Drift detection should run with read-only permissions
- Sensitive resources (security groups, IAM) should have stricter drift policies
- Consider implementing Change Advisory Board (CAB) for production changes

## Production Considerations

- **47 resources across 3 environments** — prioritize by blast radius
- Drift remediation should be tested in staging first
- Coordinate with application teams before reverting changes
- Cost: AWS Config rules cost $0.003/config rule/evaluation
- Consider using Terraform Cloud for centralized drift detection
- Implement change freeze periods during critical business events

## Senior-Level Answer

"I'd implement a 3-layer drift management strategy: (1) Prevention — restrict console access via SCPs and IAM, require all changes through Terraform with PR review, (2) Detection — automated weekly `terraform plan` scans with AWS Config for continuous monitoring, (3) Remediation — categorized workflow where intentional drift updates code, accidental drift reverts to state, and unknown drift gets investigated. The key insight is that drift is a symptom of process failure — the root fix is organizational, not technical."

## Architect-Level Answer

"At an organizational level, I'd establish a Zero Drift Policy where all infrastructure changes must go through IaC pipelines. This requires: (1) SCPs blocking manual console changes for production resources, (2) A break-glass procedure for emergencies with mandatory post-incident IaC updates tracked as tech debt, (3) Drift detection as a compliance requirement integrated into audit processes, (4) Organizational culture shift — treat infrastructure changes like code changes (PR review, testing, approval), (5) Implement Infrastructure as Code Center of Excellence to drive adoption and standards."

## Follow-Up Questions

1. "How do you handle drift detection when using Terraform modules with `for_each` or `count`?"
2. "What's the difference between drift and configuration divergence in a multi-team environment?"
3. "How do you handle drift when using Terraform Cloud with remote state and VCS integration?"
4. "How would you implement drift detection for Kubernetes resources managed by Terraform?"
5. "What's your approach to handling drift in a environment where some resources are managed by Terraform and others by CloudFormation?"
