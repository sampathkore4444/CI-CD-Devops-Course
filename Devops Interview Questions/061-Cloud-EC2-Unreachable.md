# 61. EC2 Instance Unreachable - Troubleshooting

## Scenario

A critical microservice runs on a single EC2 instance in a private subnet. At 9:15 AM, the monitoring dashboard shows the instance is no longer reporting metrics. The Service Health API starts returning 503. The instance IP is unreachable from the application tier. Other instances in the same subnet are responding fine, which rules out a subnet-wide network issue. The instance is in `running` state but `StatusChecks` show `1/2 checks passed` (instance status OK, system status FAILED, or similar). You need to recover this instance or fail over to a backup — but you can't even SSH into it to see what's happening.

## Interviewer Question

"An EC2 instance in a private subnet is unreachable — you can't SSH in, status checks are failing, but the instance is still running and other instances in the subnet are fine. Walk me through the full troubleshooting process using the AWS console, CLI, and understanding of VPC mechanics."

## What I Should Think About

- EC2 unreachable has many layers: instance OS, guest network, hypervisor, subnet, security groups, NACL, route tables, IAM permissions
- Status checks are separate: System status check (AWS infrastructure) vs Instance status check (guest OS)
- Since other instances in the same subnet work, subnet routing, NACLs, and internet gateway are probably fine
- Scope it: is it the specific instance's SG, its OS, its hypervisor, or its network?
- AWS console Instance Status / System Log / Screenshot tell you a lot without SSH
- SSM Session Manager can give you a shell even without SSH
- Stop/Start changes the underlying host (fixes some system issues) — risky if EBS is not encrypted or instance store
- Check if it's an instance store volume case (data loss on stop/start) — use EBS-backed for this reason
- Look at the instance metadata, CloudWatch metrics, and logs
- Check security groups for that instance specifically vs others
- Check the ENI, route table association, source/dest check
- Reboot vs Stop/Start — reboot is gentler, stop/start moves the hypervisor

## Ideal Answer

**Phase 1 — Confirm scope with console**

1. Check EC2 instance state: `running` (or `stopped`, or `terminated`)
2. Check status checks: `2/2 passed` vs `1/2` or `0/2`
   - System status check failed → problem is AWS side (host, network, power)
   - Instance status check failed → guest OS issue (kernel, filesystem, network config)
3. Check CloudWatch metrics for the instance: CPUUtilization, NetworkIn, NetworkOut, StatusCheckFailed
4. Check System Log (instance console output) and Get Screenshot for guest OS state
5. Check if the instance has a public IP / is in a public subnet vs private

**Phase 2 — Network troubleshooting from AWS side**

```bash
# Instance status
aws ec2 describe-instance-status --instance-ids i-0abc123def456789 \
  --query 'InstanceStatuses[0].{Status:InstanceStatus.Status, SystemStatus:SystemStatus.Status}'

# Get console output
aws ec2 get-console-output --instance-id i-0abc123def456789 --output text

# Get screenshot as base64 (view JSON or decode)
aws ec2 get-console-screenshot --instance-id i-0abc123def456789

# Check Security Groups — compare with a working instance
aws ec2 describe-instances --instance-ids i-0abc123def456789 \
  --query 'Reservations[0].Instances[0].SecurityGroups'

# Check network interfaces
aws ec2 describe-network-interfaces \
  --filters Name=attachment.instance-id,Values=i-0abc123def456789

# Check route table for the private subnet
aws ec2 describe-route-tables \
  --filters Name=association.subnet-id,Values=subnet-0xxxxxxxxx

# Check if source/destination check is enabled
aws ec2 describe-instances --instance-ids i-0abc123def456789 \
  --query 'Reservations[0].Instances[0].NetworkInterfaces[0].SourceDestCheck'
```

**Phase 3 — Try to gain access without SSH**

If the instance has an IAM role with SSM access, use SSM Session Manager. This bypasses SSH and uses the SSM agent over the private network:

```bash
# Check if SSM agent is running
aws ssm describe-instance-information \
  --filters Key=InstanceIds,Values=i-0abc123def456789

# Start a session
aws ssm start-session --target i-0abc123def456789
```

If SSM not available, use a rescue instance approach:
1. Detach the EBS root volume
2. Attach it to a working instance
3. Mount and inspect/repair filesystem or logs

**Phase 4 — Recovery**

```bash
# Reboot (gentle, keeps same host)
aws ec2 reboot-instances --instance-ids i-0abc123def456789

# If reboot doesn't fix it, stop and start (moves to new host)
aws ec2 stop-instances --instance-ids i-0abc123def456789
aws ec2 start-instances --instance-ids i-0abc123def456789

# If AMI backup available, launch replacement and update DNS/LB target
aws ec2 run-instances --image-id ami-0xxxxxxxx --instance-type t3.large ...
```

## Architecture

```
    EC2 Unreachable - Diagnostic Flow
    ─────────────────────────────────

    ┌───────────────────────┐
    │ Instance Unreachable  │
    └──────────┬────────────┘
               │
               ▼
    Check Instance State
    ┌──────────────────┐  stopped/terminated  ┌──────────────┐
    │ Running (1/2)    │─────────────────────▶│ Start instance  │
    └────────┬─────────┘                      └──────────────┘
             │ running
             ▼
    Check Status Checks
    ┌─────────────────────┐
    │ System check FAILED  │──▶ AWS host/network issue → Stop/Start (new host)
    ├─────────────────────┤
    │ Instance check FAILED│──▶ Guest OS issue → Console log / rescue
    └──────────┬──────────┘
               │
               ▼
    Network Layer Checks
    ├── Security Group matches working instance?
    ├── Subnet route table OK?
    ├── NACL OK?
    ├── Source/Dest check disabled?
    │
    ▼
    Access Bypass Options
    ├── SSM Session Manager (in-band, no SSH)
    ├── EC2 Serial Console (T2/T3, needs config)
    ├── Rescue: detach root volume → attach to healthy instance → fix OS
    │
    ▼
    Final Recovery
    ├── Reboot
    ├── Stop/Start (new hypervisor)
    ├── Launch replacement from AMI + swap in LB
    └── Restore from snapshot if data loss
```

## Investigation

1. **Confirm instance state** — it might be stopped or terminated (someone stopped it)
2. **Check status checks** — determines if it's AWS-side or guest OS side
3. **Compare security groups** with a working instance in the same subnet
4. **Check NACL and route table** — but since peers work, focus on instance-specific stuff
5. **Check the ENI** — is it attached, does it have the right private IP, is source/dest check off?
6. **Check the instance's IAM role** — can it reach SSM/System Manager for a rescue session?
7. **Get console output** — kernel panics, filesystem errors show up here
8. **Get a screenshot** — see the actual guest console
9. **Check CloudWatch alarms/metrics** — was there a CPU/memory spike before failure?
10. **Check CloudTrail** — did someone stop/reboot/modify the instance or SG recently?
11. **Check EBS volume status** — is the root volume degraded or impaired?
12. **Check AWS service health** — occasionally regional issues affect specific AZs/hosts

## Commands

```bash
# Status checks
aws ec2 describe-instance-status --instance-ids i-0xxxx --include-all-instances

# Detailed instance info
aws ec2 describe-instances --instance-ids i-0xxxx --output json

# Console output (guest log)
aws ec2 get-console-output --instance-id i-0xxxx --output text --no-paginate

# Screenshot
aws ec2 get-console-screenshot --instance-id i-0xxxx --output json

# Recent events via CloudTrail
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceName,AttributeValue=i-0xxxx \
  --start-time "$(date -u -d '4 hours ago' +%Y-%m-%dT%H:%M:%SZ)"

# Security group comparison
aws ec2 describe-security-groups \
  --group-ids $(aws ec2 describe-instances --instance-ids i-0xxxx \
    --query 'Reservations[0].Instances[0].SecurityGroups[].GroupId' --output text)

# Try SSM
aws ssm describe-instance-information --filters "Key=InstanceIds,Values=i-0xxxx"
aws ssm start-session --target i-0xxxx

# EBS volume status
aws ec2 describe-volumes --filters Name=attachment.instance-id,Values=i-0xxxx \
  --query 'Volumes[0].{State:State,Status:Status,Size:Size}'

# Stop/start recovery
aws ec2 stop-instances --instance-ids i-0xxxx
aws ec2 start-instances --instance-ids i-0xxxx

# Rescue: detach & reattach root volume
aws ec2 detach-volume --volume-id vol-0xxxx
aws ec2 attach-volume --volume-id vol-0xxxx --instance-id i-healthy --device /dev/xvdf
```

## Root Cause

| Root Cause | Evidence | Fix |
|---|---|---|
| Guest OS hung/panic | Instance status check failed, console output shows kdump/panic | Reboot; rescue mount to inspect |
| Filesystem full | Cannot start services, app logs show "no space left" | Resize/cleanup via rescue mount |
| Memory exhaustion (OOM) | System log shows "out of memory" | Increase memory or trigger stop/start |
| AWS host issue | System status check failed | Stop/Start to move to new host |
| EBS volume degraded | Volume status impaired (CloudWatch) | Restore from snapshot |
| SG changed | SG rules don't allow incoming from app tier | Roll back SG change |
| Network misconfig in guest | Instance check failed, network services down | Rescue mount, fix /etc/sysconfig/network-scripts |
| Security attack | Locked out by brute force / crypto mining | Launch from clean AMI, harden |
| Source/dest check enabled wrongly | Instance not passing NAT'd traffic | Disable on NAT-like instances only |
| Accidental stop by teammate | CloudTrail shows StopInstances | Restart, add permission guardrails |

## Immediate Mitigation

1. **Reboot first** (two minutes, least invasive): `aws ec2 reboot-instances`
2. **If system check failed**, Stop/Start (moves to a new host): `aws ec2 stop-instances; aws ec2 start-instances`
3. **If it's an AWS-side issue and stop/start fails**, launch a replacement from the latest AMI and update the target group / DNS:
   ```bash
   aws ec2 run-instances --image-id ami-latest --instance-type t3.large ...
   aws elbv2 register-targets --target-group-arn <arn> --targets Id=<newec2>
   ```
4. **Deregister the bad instance** from the ALB target group so we don't route to it while it's broken:
   ```bash
   aws elbv2 deregister-targets --target-group-arn <arn> --targets Id=i-0xxxx
   ```
5. **If SSM works**, use Session Manager to run commands to restart services without SSH.

## Permanent Fix

1. **Move microservices to auto-scaling** — never run critical services on a single instance:
   - ASG with min=2, max=10 across AZs
   - CloudWatch health checks, replace unhealthy instances automatically
2. **Standardize on EBS-backed instances** with automated snapshots + AMI lifecycle:
   - Daily AMI backup, 7-day retention via AWS Backup
   - Enable EBS Fast Snapshot Restore for the app volumes
3. **Enable SSM Session Manager everywhere** so you always have a shell without SSH keys
4. **Add status check alarm → auto-recovery**:
   ```bash
   aws cloudwatch put-metric-alarm \
     --alarm-name EC2-Recover-<instance> \
     --metric-name StatusCheckFailed_System \
     --namespace AWS/EC2 --statistic Maximum \
     --period 60 --evaluation-periods 2 \
     --threshold 1 --comparison-operator GreaterThanThreshold \
     --dimensions Name=InstanceId,Value=i-0xxxx \
     --alarm-actions arn:aws:automate:us-east-1:ec2:recover
   ```
5. **Enable EC2 Auto-Recovery** (reboots on same host vs. stop/start new host policy)
6. **Set up EC2 Serial Console** for T2/T3/C5/M5 instance classes as an emergency console
7. **Harden OS** — limit SSH source IPs, fail2ban, disable password auth

## Monitoring

```yaml
# CloudWatch alarms
- alert: EC2StatusCheckFailed_System
  expr: AWS/EC2 StatusCheckFailed_System > 0
  for: 1m
  action: auto-recovery + page on-call

- alert: EC2StatusCheckFailed_Instance
  expr: AWS/EC2 StatusCheckFailed_Instance > 0
  for: 1m
  action: page on-call, alert to create SSM session

- alert: HighCPU
  expr: AWS/EC2 CPUUtilization > 90%
  for: 15m
  action: investigate for crypto-mining/lockup

# Also monitor (requires CloudWatch Agent)
#   - memory usage
#   - disk usage
#   - per-process metrics
```

## Security

- **Don't put SSH open to 0.0.0.0/0** — use bastion/jump host or SSM only
- Use SSM Session Manager instead of SSH keys — short-lived, auditable sessions
- **Block port 22 from the internet** via SG; restrict to VPN/CIDR
- Private subnets for app/db — no public IPs on critical instances
- Enable **IMDSv2** and restrict to token-required
- Instance IAM role with least privilege — app role shouldn't have EC2 admin
- Encrypt EBS volumes with KMS customer-managed keys
- Run on current AMI versions — patch monthly, watch for CVEs (e.g., Log4j)
- GuardDuty to detect brute force/crypto-mining and trigger auto-remediation

## Production Considerations

- **Cost**: Single instance is cheap but produces expensive downtime — right size and use 2+ instances
- **HA**: Run 2+ instances behind ALB in different AZs; ASG min=2
- **Reliability**: Golden AMI pipeline (Packer) for fast, reproducible replacement
- **Operational**: Use Instance Metadata Service (`curl http://169.254.169.254/latest/meta-data/`) to verify instance identity during debugging
- **Compliance**: Snapshot retention policy for audit; CloudTrail for all modifications
- **Multi-region**: For true DR, replicate AMI and consider ASG in a second region
- **Backup**: AWS Backup with scheduled snapshots, lifecycle to delete old snapshots
- **Data**: If using instance store, accept data loss on stop/start; keep critical data in EBS/RDS/S3

## Senior-Level Answer

"I scope it before touching anything. If other instances in the subnet work, the problem is instance-specific — either the guest OS, the instance's own SG/ENI, or the AWS host. I check status checks: system check failure points to AWS-side (fix: stop/start to move hosts), instance check failure points to guest OS. I pull the console output and screenshot to see kernel panics or a hung boot. I check CloudTrail to see if someone stopped/modified it. For access, I prefer SSM Session Manager over SSH — it works even without network reachability. If the OS is wedged, I detach the root volume, attach it to a healthy instance, and repair or extract data. For recovery, reboot → stop/start → restore from AMI/snapshot, in that order. And critically, a single instance serving a critical service is an architectural problem — I'd push to move it behind an ASG."

## Architect-Level Answer

"Unreachable single instances are a symptom of an architecture that treats compute as pets. The strategic fix is to make every workload ephemeral: ASG-managed instances with a golden AMI pipeline, ELB health check routing, and CloudWatch-driven health replacement. For operational access, I'd standardize on SSM Session Manager (no standing SSH ports) and enable EC2 Serial Console as a plane-B console. I'd add CloudWatch auto-recovery and a failover path to launch from the latest AMI automatically. The debugging flow itself is standard: state → status checks → console output → SG/NACL/route comparison → CloudTrail → rescue mount. The architecture lesson is that instance-level reliability is table stakes — the real goal is fleet-level self-healing with no single point of failure."

## Follow-Up Questions

1. "When would you choose to reboot vs. stop/start an EC2 instance, and what are the risks of each?"
2. "A user can SSH to instance A but not to instance B in the same subnet, same SG. What are the top 3 reasons?"
3. "How do you demonstrate that a security group was the root cause?" 
4. "What's the difference between an instance status check and a system status check, and which one requires the guest OS to be up?"
5. "If you must recover this instance's filesystem after a kernel panic, walk through the detach-and-mount rescue with exact commands."