# 64. Auto Scaling Group Not Scaling Properly

## Scenario

Your company launched a Black Friday sale. Traffic has grown 3x in the last hour. The web application is served by an Auto Scaling Group with a target-tracking scaling policy based on average CPU at 50%. Users are hitting 503 timeouts. The ASG should have scaled out — the average CPU across instances is at 80%. But the desired capacity is still 2, the same as before the spike. You can see the EC2 instances underutilized at high CPU and the load balancer overwhelmed. Your job is to figure out why the ASG isn't scaling and get capacity up before the sale window passes.

## Interviewer Question

"Traffic tripled but the Auto Scaling Group didn't add instances and now users are timing out. Walk me through diagnosing ASG scaling failures and the fix — including CloudWatch metrics, health checks, policies, and network/instance factors."

## What I Should Think About

- Autoscaling depends on a chain: metric → CloudWatch alarm → policy → ASG capacity change. Any break means no scale-out
- Target-tracking uses pre-defined CloudWatch alarms and its own metric — if the metric has no data (e.g., average CPU NaN), it won't trigger
- CloudWatch alarm **ALARM state** triggers the default 300s cool-down; then instances launch. There's inherent delay (metric period + evaluation × 2-3)
- SCT (Scheduled cCapacity) override: scheduled actions beat dynamic scaling and may cap capacity
- Check that the ASG has valid SGs and that the LB target group registration health isn't broken
- If instances launch but are terminated immediately (health check failed / not healthy), it looks like "not scaling"
- Instance launch limits (max instances per region, per instance type) can silently block scaling
- IAM: the ASG needs `autoscaling:*` and `ec2:RunInstances` permission
- CloudWatch metric alarm status might be INSUFFICIENT_DATA due to missing metrics (per-second vs aggregated)
- Scaling via **Target Tracking** uses the same alarm for both in/out — misconfigured metric → no triggers
- Check the ASG's **cooldown** and **default instance warmup**, and the **maximum size** is greater than desired
- AWS budget/limits errors come as "LimitExceeded" in history

## Ideal Answer

**Step 1 — Confirm ASG state and get scaling history**

```bash
# ASG config + capacity
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names web-app-asg \
  --query 'AutoScalingGroups[0].{Desired:DesiredCapacity,Min:MinSize,Max:MaxSize,Instances:Instances}'

# Scaling activities (shows why / why not)
aws autoscaling describe-scaling-activities --auto-scaling-group-name web-app-asg \
  --query 'Activities[0:5].{Description:Description,Cause:Cause,StatusCode:StatusCode,Details:Details}'

# Policies
aws autoscaling describe-policies --auto-scaling-group-name web-app-asg \
  --query 'ScalingPolicies[]'

# Scheduled actions (may override)
aws autoscaling describe-scheduled-actions --auto-scaling-group-name web-app-asg
```

**Step 2 — Check CloudWatch metric + alarm**

```bash
# Target-tracking policy creates alarms; check them
aws cloudwatch describe-alarms \
  --alarm-name-prefix "Web-App" --output table

# Look at the actual metric values (CPU)
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 --metric-name CPUUtilization \
  --dimensions Name=AutoScalingGroupName,Value=web-app-asg \
  --start-time "$(date -u -d '30 min ago' +%Y-%m-%dT%H:%M:%SZ)" \
  --end-time "$(date -u)" --period 60 --statistics Average

# Is the alarm firing?
aws cloudwatch describe-alarm-history \
  --alarm-name "TargetTracking-web-app-asg-AlarmHigh-..." \
  --history-item-type StateUpdate
```

**Step 3 — Check if instances are actually launching, then dying**

```bash
# Instance states inside ASG
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names web-app-asg \
  --query 'AutoScalingGroups[0].Instances[*].{Id:InstanceId,Health:HealthStatus,Lifecycle:LifecycleState}'

# Recent launches
aws ec2 describe-instances \
  --filters Name=tag:aws:autoscaling:groupName,Values=web-app-asg \
  --query 'Reservations[].Instances[].{Id:InstanceId,State:State.Name,InstType:InstanceType}'

# Target group health
aws elbv2 describe-target-health --target-group-arn <arn> --output table
```

If instances are launched but then terminated, look at:
- ELB health check failures (app not starting in time)
- Instance shutdown (Spot interruption)
- Health check grace period too short

**Step 4 — Fix**

Depending on cause, some fixes:
```bash
# Force scale-out NOW to relieve pressure
aws autoscaling update-auto-scaling-group --auto-scaling-group-name web-app-asg \
  --desired-capacity 8 --max-size 12

# Lower cool-down so it scales faster
aws autoscaling update-auto-scaling-group --auto-scaling-group-name web-app-asg \
  --default-cooldown 120

# Increase max size if it was the cap
aws autoscaling update-auto-scaling-group --auto-scaling-group-name web-app-asg --max-size 16

# If alarm was INSOMNIA, pick a better metric/policy (e.g., based on ALB requests/minute)
# or use a custom metric per-minute
```

## Architecture

```
    Autoscaling chain — where it can break
    ──────────────────────────────────────

    Traffic ↑  ──▶  EC2 CPU stays high
                      │
                      ▼
              CloudWatch metrics (EC2, 1-min)
                      │  (metric missing? INSUFFICIENT_DATA)
                      ▼
              CloudWatch Alarm (TargetTrackingHigh)
                      │  (periods × evaluation period delay)
                      ▼
              Scaling Policy (target-tracking)
                      │  (cooldown / warmup)
                      ▼
              ASG DesiredCapacity += n
                      │  (max size cap? scheduled actions? limits)
                      ▼
              EC2 launch → register to ALB TG
                      │  (health check pass?)
                      ▼
              Traffic begins to flow
```

## Investigation

1. **Check ASG desired vs actual capacity** — if desired rose but no instances, it's a launch problem
2. **Review scaling activities** — the `Cause` string explains what triggered (or blocked) the action
3. **Check the CloudWatch alarm state** — OK/ALARM/INSUFFICIENT_DATA
4. **Check the metric quality** — is average CPU populated? Are there NaN gaps (per-second enabled)
5. **Check policies** — target-tracking may be misconfigured or the wrong metric
6. **Check scheduled actions** — they can hold desired capacity at a fixed value
7. **Check instance launch errors** — `LimitExceeded`, insufficient capacity in AZ, Spot issues
8. **Check target group health** — launched instances, but unhealthy → terminated → "not scaling"
9. **Check cooldown and warmup** — too long = slow response to spikes
10. **Check IAM/service-linked role** — ASG must have permission to launch instances

## Commands

```bash
# Full ASG details
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names web-app-asg \
  --query 'AutoScalingGroups[0].{Capacity:DesiredCapacity,Min:MinSize,Max:MaxSize,LaunchTemp:LaunchTemplate.LaunchTemplateName,Cool:DefaultCooldown,TerminationPolicies:TerminationPolicies}'

# Scaling policy + alarm linkage
aws autoscaling describe-policies --auto-scaling-group-name web-app-asg \
  --output json | jq '.ScalingPolicies[] | {Name:AlarmName,Type:PolicyType,Target:TargetTrackingConfiguration.TargetValue,ARN:PolicyARN}'

# Alarm evaluation history
aws cloudwatch describe-alarms \
  --query 'MetricAlarms[?contains(AlarmName,`TargetTracking-web-app-asg`)].{Name:AlarmName,State:StateValue,Type:ComparisonOperator}' \
  --output table

# Metric for the ASG as a whole
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 --metric-name CPUUtilization \
  --dimensions Name=AutoScalingGroupName,Value=web-app-asg \
  --statistics Average --period 300 \
  --start-time "$(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ)" --end-time "$(date -u)"

# Check instance health in the TG
aws elbv2 describe-target-health --target-group-arn arn:aws:elasticloadbalancing:us-east-1:xxx:targetgroup/web-app/<id>

# Look for errors in CloudTrail (autoscaling / ec2)
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=RunInstances \
  --start-time "$(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ)"
```

## Root Cause

| Root Cause | Evidence | Fix |
|---|---|---|
| Max size reached | desired == max, alarm firing but no capacity change | Raise `MaxSize`, enable instance type limits review |
| Metric missing/NaN | CPUUtilization INSUFFICIENT / no data | Enable per-second metrics or use ALB request metric |
| Wrong metric in policy | Target 50% CPU but metric is per-instance | Use ASG average CPU or request-count-based policy |
| Too-long cooldown | Activities stuck in CoolDown state | Reduce `DefaultCooldown` / add `InstanceWarmup` |
| Scheduled action capping | Scheduled action sets Min/Max/Desired fixed | Remove/update scheduled action |
| Instances launch then die | Health check fails on TG; iteration loops | Fix app startup health path / increase grace period |
| Instance limit / Spot capacity | `LimitExceeded`, insufficient capacity errors | Request limit increase, use mixed-instance pool |
| Bad launch template / IP | Launch fails at `RunInstances` (missing SG/subnet) | Fix launch template subnets across AZs |
| IAM missing around autoscaling | Permission errors in CloudTrail | Attach `AmazonEC2FullAccess`-equivalent, service-linked role |
| Stuck alarm (stale evaluation) | Alarm in OK despite high metric | Fix metric resolution / alarm range |

## Immediate Mitigation

1. **Manually raise desired capacity immediately** to relieve the traffic (time-to-value < 1 second through API):
   ```bash
   aws autoscaling update-auto-scaling-group \
     --auto-scaling-group-name web-app-asg \
     --desired-capacity 8 \
     --max-size 16
   ```
2. **Also add a burst of capacity in the AZ** if the ALB is overwhelmed, or front it with a shock absorber (rate limiter / 503 page with queue depth).
3. **If instances charge but die**, check ASG health and remove unhealthy targets; fix the health path.
4. **Temporarily raise per-region/type limits** if `LimitExceeded` in the middle of the spike (autoscaling can't launch without quota headroom).
5. **Stop the 503 storm**: consider a warm CloudFront cache hit-rate increase so origin load drops while ASG catches up.

## Permanent Fix

1. **Use target-tracking with a good metric** — prefer `ALBRequestCountPerTarget` (survives AZ imbalance) or ASG average CPU. Set target 50-60%.
2. **Add a second, faster scale-out** using heartbeat metric (e.g., ASG CPU at 1-minute) with lower alarm evaluation periods (1/1) and short cooldown.
3. **Scale based on queue depth** when the app is backend-heavy (e.g., SQS depth / LB pending requests).
4. **Turn on predictive scaling** (AWS Predictive Scaling) for seasonal spikes to pre-warm capacity.
5. **Mixed instances / capacity-optimized allocation** across AZs to survive capacity issues.
6. **Periodic load tests** (e.g., Load Impact / k6) and validate scale-out metrics drive 2x capacity within N minutes.
7. **DefaultInstanceWarmup** to avoid alarm flapping after a launch.

## Monitoring

```yaml
# Alarm for ASG health
- alert: ASGDesiredVsRunningMismatch
  expr: aws_autoscaling_group_instances != aws_autoscaling_group_desired
  for: 5m
  severity: critical

- alert: ScalingActivityFailed
  expr: autoscaling:ActivityFailed log event
  for: 3m
  severity: critical

- alert: ASGMaxCapacityReached
  expr: aws_autoscaling_group_desired > 0.95 * aws_autoscaling_group_max
  for: 10m
  severity: warning
```

Enable ASG **CloudWatch detailed metrics** for per-instance visibility and faster alarm evaluation.

## Security

- Scope ASG IAM tightly (service-linked role `AWSServiceRoleForAutoScaling`)
- Instance role launched by ASG uses least-privilege (no `ec2:CreateSecurityGroup` from app instances)
- Vary launch config to keep instances in private subnets, no public IP unless required
- Rotate AMIs; scan images; use `Imdsv2TokenRequired` 
- Monitor for crypto-mining (high CPU on idle scale-out cycles) via GuardDuty

## Production Considerations

- **Cost**: Right-size instance types in launch template; Spot for stateless burst workloads (with Capacity-optimized allocation); enforce ASG plan = max size you can afford
- **HA**: multi-AZ spread; health checks via ALB with registration delays and warmup
- **Reliability**: cooldown + warmup tuned so healthy instances aren't flushed during rolling deploys
- **Operational**: keep a runbook for scaling failures; alert on `avoid.other` activities, never page raw `any node unhealthy`
- **Compliance**: ASG modifications via IaC; access audit in CloudTrail
- **DR**: multi-region ASG with health-check routing (Route53 weighted failover)

## Senior-Level Answer

"First, I check whether the ASG attempted anything: `describe-scaling-activities` tells me if it tried and failed or never got a signal. Then I verify the watchdog chain — metric, alarm state, policy — because target-tracking depends on a CloudWatch alarm that can sit in INSUFFICIENT_DATA. I'd check the metric's averaging: per-second metrics often produce NaN for instance-level CPU. If the alarm fires but capacity is capped, it's `MaxSize`, a scheduled action, or a quota limit. If instances launch and die, it's an ALB health-check loop and looks like 'no scaling'. For the black-friday scenario, the fix is to raise desired capacity immediately, widen MaxSize, and add a longer warmup + lower cooldown, then move to a request-count-based scale policy and predictive scaling for seasonality."

## Architect-Level Answer

"Scaling is a feedback loop, and robustness means the loop must not have hidden single points of failure. I design the loop around ASG average CPU or ALB request count with target-tracking, plus a fast secondary policy (SQS depth / LB pending requests) and predictive scaling for seasonality. I enforce per-step grace, warmup, and cooldown; I monitor both the metric and the activity stream; I alert on mismatch of desired vs running. I also apply the 'capacity headroom' principle — MaxSize at least 2-3x expected peak so dynamic scaling has room. For flash traffic this fails; put a CDN cache in front and consider rate-limiting, plus capacity-optimized mixed instances to survive instance-type shortages. The architecture lesson: never let scaling mechanisms block availability — the ASG is the control loop, and the plan must be verified by load tests at least quarterly."

## Follow-Up Questions

1. "Target-tracking scaling uses a CloudWatch alarm that can show INSUFFICIENT_DATA. Explain why per-second metric resolution matters and how it changes the alarm behavior."
2. "What's the difference between an ASG cooldown period and the default instance warmup? What happens if you set warmup too low?"
3. "An instance healthy status at the ALB is 'unhealthy', and the ASG kills it repeatedly. Walk through the diagnosis — where would you see the health check failures and what's the fastest fix?"
4. "A scheduled action sets desired capacity to 10, but traffic drops. Why is the ASG at 10 and not scaling down? How would you handle this difference between 'standby' and 'desired'?"
5. "Design a scaling strategy for a 5x seasonal spike using predictive scaling, target-tracking, and mixed instances. Where do you place Spot vs On-Demand?"