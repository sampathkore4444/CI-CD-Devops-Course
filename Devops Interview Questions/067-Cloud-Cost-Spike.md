# 67. AWS Bill Spike - Cost Optimization

## Scenario

It's the end of the month and the finance team sends a Slack message: "The AWS bill went up 400% this month compared to last month. This wasn't in the budget. We need to know what caused this and how to prevent it from happening again — ideally by the end of the day." The infrastructure includes EC2 (EKS nodes), RDS, S3, CloudFront, NAT Gateway, and significant data transfer. Nobody did a large intentional change. You need to analyze Cost Explorer, find the spike, identify the offending resources, and take action.

## Interviewer Question

"Your AWS bill spiked 400% in one month. Walk me through the investigation to identify what's driving cost, how you'd verify it's legitimate, and how you'd implement ongoing cost optimization."

## What I Should Think About

- Use **AWS Cost Explorer** and **Cost and Usage Report (CUR)** for granular breakdown
- Get to the bottom-up: month-over-month by service, by usage type, by region, by tag
- Check for schema changes that shift cost categories (e.g., a service started using different usage types)
- Data transfer charges are sneaky — usually not top-line visible
- NAT Gateway charges: per GB processed + hourly — a misbehaving EKS/ECS app can blow up NAT egress
- EKS autoscaling failures → huge node fleet at idle
- S3: `Get`/`Lifecycle`?? or replicas
- EC2 running 24/7 orphaned (ASG mismatch, stopped-but-charged volumes, EIP)
- RDS on expensive instance type or idle read replica
- Check CloudFront data transfer + `Invalidation` charges
- Check Public IPv4 address charges (AWS introduced them in 2024 — cost for any public IPv4)
- Look for database storage I/O spikes (cold start)
- Check GPU/spot price changes, or unguarded new region deployments
- **Tagging strategy** — resources not tagged make attribution hard
- Cost is Lead Time backwards: use Cost Anomaly Detection — it should have caught this earlier — for future
- Use **Budget + Budget Alerts** with actual/forecast tracking
- Look at **Trusted Advisor** / **Compute Optimizer** for right-sizing recommendations
- Use **AWS RIs/Savings Plans** marketplace sudden spend?

## Ideal Answer

**Step 1 — Get the bill breakdown (Cost Explorer)**

```bash
# Month-over-month by service
aws ce get-cost-and-usage \
  --time-period Start=$(date -u -d "$(date +%Y-%m)1" +%Y-%m-01),End=$(date -u -d "$(date -d "$(date +%Y-%m-15)" +%Y-%m-20) +1 month" +%Y-%m-01) \
  --granularity DAILY \
  --metrics "UnblendedCost" \
  --group-by Type=DIMENSION,Key=SERVICE \
  --filter '{"Dimensions":{"Key":"LINKED_ACCOUNT","Values":["<acct>"]}}' \
  --output table
```

**Step 2 — Find which resource & usage type**

```bash
# Group by usage type + instance type (SEV)
aws ce get-cost-and-usage \
  --time-period ... --granularity DAILY \
  --metrics "UnblendedCost" \
  --group-by Type=DIMENSION,Key=USAGE_TYPE

# Group by tag/linked account/resource
aws ce get-cost-and-usage \
  --time-period ... --granularity MONTHLY \
  --metrics "UnblendedCost" \
  --group-by Type=TAG,Key=Environment

# Historical comparison of the same period
aws ce get-cost-and-usage \
  --time-period ... --granularity DAILY \
  --metrics "UnblendedCost" \
  --group-by Type=DIMENSION,Key=SERVICE \
  --output table
```

**Step 3 — Drill into specific suspects**

```bash
# EC2/data transfer/NAT by usage type
aws ce get-cost-and-usage \
  --group-by Type=DIMENSION,Key=SERVICE \
  --filter '{"Dimensions":{"Key":"USAGE_TYPE","Values":["NatGateway-Bytes","DataTransfer-Out-Bytes","EBS:VolumeUsage","S3-Out-Bytes"]}}'
```

**Step 4 — Verify and remediate**

- If NAT egress: use VPC Flow logs to quantify traffic; identify app doing loops/misconfigured webhooks
- If EC2: check for orphaned instances (ASG scaled down but old instances left, or the billing for stopped volumes, EIPs)
- If RDS: check read replica idle, wrong size, or increased IOPS
- If S3: check GET requests via storage metrics; enable lifecycle
- If transfer: examine `DataTransfer-Out` to ANY / `VPC-GW` entries

Remediation:
```bash
# Stop an idle stray instance
aws ec2 stop-instances --instance-ids i-0xxx

# Downsize a read replica (check load first!)
aws rds modify-db-instance --db-instance-identifier my-replica \
  --db-instance-class db.t3.small --apply-immediately

# Add a lifecycle rule to archive old logs
aws s3api put-bucket-lifecycle-configuration --bucket archive-bucket \
  --lifecycle-configuration '{"Rules":[{"ID":"archive","Status":"Enabled","Filter":{"Prefix":"logs/"},"Transitions":[{"Days":30,"StorageClass":"GLACIER"},{"Days":90,"StorageClass":"DEEP_ARCHIVE"}]}]}'

# Delete orphaned EIPs
aws ec2 release-address --allocation-id eipalloc-xxx
```

**Step 5 — Prevent recurrence**

- Set a monthly budget:
```bash
aws budgets create-budget --account-id 123456789012 \
  --budget '{"BudgetName":"Monthly AWS","BudgetLimit":{"Amount":"20000","Unit":"USD"},"TimeUnit":"MONTHLY","BudgetType":"COST"}' \
  --notifications '[{"Notification":{"NotificationType":"ACTUAL","ComparisonOperator":"GREATER_THAN","Threshold":80},"Subscribers":[{"SubscriptionType":"EMAIL","Address":"team@example.com"}]}]'
```
- Enable **Cost Anomaly Detection** (AWS Cost Anomaly Detector) with alerting.
- Implement tagging policy — all resources tagged for `BusinessUnit/Environment/Tenant usage` so costs map to owners.

## Architecture

```
    Cost spike — attribution flow
    ─────────────────────────────

    Billup 400% ──▶ Cost Explorer: daily spend graph
                         │
                         ▼
             Service-level breakdown (EC2, S3, NAT, RDS...)
                         │
                         ▼
             Usage-type level (CPU-Hours, NatGateway-Bytes, DB-IOPS)
                         │
                         ▼
             Tag/resource level (Env=prod, App=..., Cluster=...)
                         │
                         ▼
            Verify with: CloudWatch, Flow Logs, Trusted Advisor,
            Compute Optimizer, CloudTrail (who changed what)
                         │
                         ▼
            Remediate (stop/downsize/delete/archive) + Budget + Anomaly Detection

    Suspects to check in order:
    1. NAT Gateway bytes (egress explosion)
    2. Orphaned/oversized EC2 (EKS node count, idle FE fleet)
    3. RDS size/replicas/IOPS
    4. S3 GET/lifecycle + Glacier restore cost
    5. Data transfer / CloudFront
    6. Public IPv4 charges
    7. Reserved/Savings Plans changes
```

## Investigation

1. **Monthly & daily trend** — which day did it jump?
2. **Service breakdown** — which service grew?
3. **Usage type drill** — is it compute time, data transfer, storage?
4. **Tag analysis** — which team/application/environment owns it
5. **Region analysis** — did something deploy to a new region?
6. **Cross-reference CloudTrail** — resource launches/changes in that window (RunInstances, CreateDBInstance, PutBucketLifecycle...)
7. **Verify suspicious resources in console** — is that instance legitimate?
8. **Check for crypto-mining** — high CPU + transfer out (GuardDuty)
9. **Check CloudFront metrics** — unexpected GET/Invalidation
10. **Validate the numbers** — Cost Explorer's unblended vs actual after Payer discounts; and confirm the 400% is real vs. accrual timing

## Commands

```bash
# 1. Daily cost by service (last 30d)
aws ce get-cost-and-usage \
  --time-period Start="$(date -u -d '30 days ago' +%Y-%m-%d),End=$(date +%Y-%m-%d)" \
  --granularity DAILY --metrics UnblendedCost \
  --group-by Type=DIMENSION,Key=SERVICE

# 2. Which usage type exploded
aws ce get-cost-and-usage \
  --time-period ... --granularity DAILY --metrics UnblendedCost \
  --group-by Type=DIMENSION,Key=USAGE_TYPE

# 3. Tag/ownership
aws ce get-cost-and-usage \
  --time-period ... --granularity MONTHLY --metrics UnblendedCost \
  --group-by Type=TAG,Key=App

# 4. Histogram of days (day-level)
aws ce get-cost-and-usage --granularity DAILY --metrics UnblendedCost

# 5. Drill flow logs for NAT volume
aws logs filter-log-events \
  --log-group-name /aws/vpc/flowlogs/prod \
  --start-time "$(date -u -d '3 days ago' +%s)000"

# 6. Find orphaned instances
aws ec2 describe-instances --filters Name=instance-state-name,Values=running \
  --query 'Reservations[].Instances[].[InstanceId,Tags[?Key==`Name`].Value| [0],InstanceType]'
# then compare to ASG/managed-node members

# 7. Check RDS replicas & uptime
aws rds describe-db-instances --query 'DBInstances[].[DBInstanceIdentifier,DBInstanceClass,Engine,MultiAZ,DBInstanceStatus]'
```

## Root Cause

| Root Cause | Evidence | Fix |
|---|---|---|
| NAT Gateway egress explosion | `NatGateway-Bytes` usage type dominating | Identify process via Flow Logs; cap with route-based throttling / PrivateLink |
| Orphaned/oversized EC2 (EKS node drift) | Many running instances not in ASG/managed node group | ASG-managed nodes + `reconcile`; stop stragglers |
| RDS oversized/idle replica | Read replica idle (CloudWatch DBConnections=0) | Downsize or drop after load check |
| S3 GET surge / no lifecycle | High CSV/GET usage, Glacier restore costs | Enable lifecycle, use CloudFront, delay-tiering |
| Data transfer | `DataTransfer-Out` growth | Move to CloudFront, enable Cost Anomaly on network spend |
| Public IPv4 charges | New per-IP charges meet 2024 billing | Release unattached EIPs; use matching EIP count |
| Crypto-mining compromise | CPU + egress spike, GuardDuty finding | Isolate instance, kill keys, audit SG |
| Savings Plans / RI change | Plan expired/lapsed month | Renew/auto-apply Savings Plans |

## Immediate Mitigation

1. **Identify and stop the offending resource** (orphan, crypto, runaway job) — fastest path to stopping the meter:
   ```bash
   # Example: stop runaway test cluster
   aws ec2 stop-instances --instance-ids $(aws ec2 describe-instances \
     --filters Name=tag:Environment,Values=test Name=instance-state-name,Values=running \
     --query 'Reservations[].Instances[].InstanceId' --output text)
   ```
2. **If egress is the issue**, take the microservice offline (scale deployment to 0) until root-caused.
3. **Set an immediate budget alarm** (threshold 90% of month) so you see it grow, not late.
4. **Release live EIPs/volumes** that changed the bill structure.
5. **Prevent reoccurrence during investigation**: require new resources tagged/metadata until the cause is found.

## Permanent Fix

1. **Budgets + Cost Anomaly Detection** with alerts to Slack/Teams — active prevention
2. **Enforced tagging** — cost attribution: every resource is tagged with BusinessUnit/Env/Owner; use `Prohibited or Required` in SCP for major resources
3. **Periodic right-sizing** via AWS Compute Optimizer / Trusted Advisor; review quarterly
4. **Commitment plans** (Savings Plans/RIs) for baseline steady-state load to stabilize base cost
5. **Data-transfer guardrails** — CloudFront in front of S3/ALB; enable P2P-friendly caching; use PrivateLink for VPC-to-VPC to avoid NAT egress
6. **Cost CI check** — after each release, compare per-deploy spend; a regression that 10x's cost trips CI
7. **Monthly cost review meeting** with service owners + anomaly threshold (e.g., 20% of the 90-day average)

## Monitoring

```yaml
# Cost monitoring
- alert: MonthlyBudgetForecastBreach
  expr: CostExplorer forecast > 80% of budget
  for: 1d
  severity: warning → 90%: critical

- alert: CostAnomaly
  expr: Cost Anomaly Detector anomaly > $1,000
  for: immediate
  severity: warning

# Resource-level signals that correlate with cost spikes
- alert: HighEGRESS
  expr: Networking/NetworkOut total > X GB/day on any ASG
  for: 1d
  severity: warning
```

Daily CLI check for finance: script runs `aws ce get-cost-and-usage --granularity DAILY` and posts to a cost dashboard.

## Security

- Cost exposure out of control can be a **security** signal: uncompensated crypto-mining, IP theft. Use GuardDuty + VPC Flow Logs
- Limit blast radius of IAM: restrict `ec2:RunInstances` for prod except from CI; tag new resources with Owner
- Audit CloudTrail for unexpected resource creation in the spike window
- Never leave root/SECRET keys on compromised instance — they may spin up resources at your expense (crypto)
- Set **quota limits** for team/account; SCP restricting expensive services (GPU instances) to approved IAM roles
- Public IPv4 + EIP audit to avoid surprise charges and also reduce attack surface

## Production Considerations

- **Cost**: VPC/NAT egress is a top area for 10x-100x surprises — enable Cost Explorer/RDS + CUR streaming to Athena for attribution
- **Reliability**: don't kill resources rescuing cost without verifying load — check metrics first
- **Operational**: keep a "cost runbook" — fast action when budget alarm fires
- **Compliance**: CUR for finance/audit; where data egress restrictions apply (HIPAA/PCI), egress patterns matter
- **HA**: always pair cost reductions with validation that HA isn't lost (don't stop a second AZ to save $)
- **DR**: ensure replication/backups (Glacier) aren't the spike (RPO vs cost trade-off)
- **Forecasting**: use Cost Explorer forecasts, review at month start

## Senior-Level Answer

"I start with the daily trend in Cost Explorer to find the day of the jump, then drill by service and by usage type. The top suspects for surprise 400% jumps are data transfer (NAT Gateway bytes), orphaned EC2/EKS node fleets, S3 GET surge, and RDS over-provisioning. Tagging is how you get from 'unknown' to 'owner' — so if tagging is missing, Cost Explorer surfaces little. I cross-check CloudTrail for glue events that day (RunInstances, CreateDBInstance, PutBucketLifecycle). The most common fix is temporary — stop the runaway resource and confirm the meter goes back down — then permanent: budgets + Cost Anomaly Detection + enforced tagging + right-sizing review. I also remind that a cost spike can be a security event: crypto-mining shows as high CPU plus egress; GuardDuty should be checked in parallel."

## Architect-Level Answer

"Unprecedented cost is a symptom of missing control-plane instrumentation, not just accounts. I build cost into architecture the same way I build resilience: (1) Cost and Usage Report streamed to Athena or QuickSight for queryable attribution — not ClickOps-on-Cost-Explorer; (2) budgets + Cost Anomaly Detection on both actual and forecast spend with PagerDuty integration; (3) enforced tagging strategy as a contract — unwritten resources get SCP-denied creation in prod; (4) commitment planning for the steady state (Savings Plans) so only deltas are variable and anomalies surface by delta; (5) periodic architecture-driven right-sizing reviews and a data-transfer budget per service. I'd also treat steep egress as a design review trigger: NAT-heavy architectures may move to PrivateLink/CloudFront. The architectural lesson: cost must be observable in near-real-time, attributable, and bounded — otherwise one bad flag can erase a month of budget before anyone notices."

## Follow-Up Questions

1. "Walk through how to use Cost and Usage Report + Athena to find the single largest cost driver in a 400% spike — include the SQL."
2. "What are the differences between UnblendedCost, AmortizedCost, and NetUnblendedCost in Cost Explorer, and when do you use each?"
3. "You find an orphaned EKS node group running 20 c5.4xlarge for 3 weeks. Design the process to (a) prove it's orphaned, (b) safely shut it down, (c) prevent recurrence."
4. "Your S3 GET charge tripled overnight but the bucket is heavily cached through CloudFront. What are the possible causes and how do you verify each?"
5. "How do you build a cost-anomaly detection policy that doesn't drown people in alerts — specify thresholds, service exclusions, and shadow-period strategy?"