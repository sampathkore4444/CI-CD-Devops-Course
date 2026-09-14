# 66. CloudWatch Alarms Not Triggering

## Scenario

Your application serves millions of requests a day. The monitoring team set up a CloudWatch alarm: "Alert when average CPU utilization on the web autoscale group exceeds 80% for 5 minutes." During a traffic spike, CPU has been sitting at 95% for 20 minutes. You check the dashboard — the alarm state shows **OK**, not ALARM. No one has been paged. The SLO dashboard shows increased latency, but the alarm never fired. You need to figure out why the alarm isn't transitioning to ALARM and make sure the next spike actually pages someone.

## Interviewer Question

"A CloudWatch alarm configured for CPU > 80% for 5 minutes stays OK while CPU is at 95%. Walk me through troubleshooting a non-firing CloudWatch alarm and fixing the alerting."

## What I Should Think About

- An alarm only goes off when the metric's evaluation over the entire period looks back at **Period × EvaluationPeriods** — "for 5 minutes" means multiple data points over that window must exceed the threshold
- **Statistic matters**: the alarm uses the *statistic* (Average/Maximum/Minimum) over the period. If the alarm evaluates *Average* over 300s, average may be below 80 even if instantaneous is 95
- **Metric resolution**: standard 60s vs **high-resolution** (1s/5s lambda metrics) — high-res alarms have different period constraints (min 10s) and can sit in INSUFFICIENT_DATA
- **Gap / missing data**: if the metric has holes, the alarm bounces OK/INSUFFICIENT, never reaching ALARM
- **TreatMissingData**: default is `missing` → treated as "enough data" which keeps alarm in OK — a silent killer
- **Alarm actions**: the *SNS topic* might exist but the topic subscription could be paused/unconfirmed; ALSO the `alarm-actions` may not be attached at all
- **Period vs evaluation periods**: `--period 300 --evaluation-periods 1` means "average over 5 min crosses threshold once", which IS met — so check what the actual threshold/statistic are
- "80% for 5 minutes" might be implemented as `period 300, evaluations 1, threshold 80, comparison GreaterThanThreshold` — check the actual numbers
- CloudWatch alarms with **percentile statistic** (p99) vs Average
- Alarm state history: `describe-alarm-history` shows each state transition — if it never left OK, look at the metric data: the alarm reads the metric with the *same statistic*, which may differ from dashboard percentage-per-instance
- ASG CPU is per-instance; an ASG average CPU requires aggregation across instances — a single hot instance doesn't move the aggregate
- Look at the metric *namespace/dimensions* — a wrong dimension (ASG name typo, instance type filter) = no matching data = OK/INSUFFICIENT forever
- Alarm on **ALB target group** vs EC2 instance; endpoints export `AWS/ApplicationELB`
- Check IAM: monitoring role may not have `cloudwatch:PutMetricData` / the app emits metrics with wrong account
- **max-delay-based pub/sub**: SNS delivery failures don't remove the alarm, they give `Failed Delivery` in logs

## Ideal Answer

**Step 1 — Read the alarm configuration exactly**

```bash
aws cloudwatch describe-alarms --alarm-names "AppCPUHigh" \
  --output json | jq '.MetricAlarms[0] | {MetricName, Namespace, Statistic, Period, EvaluationPeriods, Threshold, ComparisonOperator, TreatMissingData, StateValue, AlarmActions}'

# Fine-grained view of the metric involved
aws cloudwatch describe-alarms --alarm-names "AppCPUHigh" \
  --query 'MetricAlarms[0].Metrics'
```

**Step 2 — Check the actual metric data matching the same statistic**

```bash
# Pull the last 30 minutes exactly as the alarm sees it (same period + statistic)
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 --metric-name CPUUtilization \
  --dimensions Name=AutoScalingGroupName,Value=web-asg \
  --statistics Average --period 300 \
  --start-time "$(date -u -d '30 min ago' +%Y-%m-%dT%H:%M:%SZ)" \
  --end-time "$(date -u)" \
  --query 'Datapoints[0:5]' \
  --output table
```

**Step 3 — Check alarm state history**

```bash
aws cloudwatch describe-alarm-history \
  --alarm-name "AppCPUHigh" \
  --history-item-type StateUpdate --max-items 20
```

If there are no transitions, the alarm is reading the metric but it never breaches — so the metric ≠ dashboard metric. If there are transitions to INSUFFICIENT_DATA, the metric has gaps.

**Step 4 — Fix the alarm**

Based on cause:
- Wrong statistic → switch to `Maximum` for "reached 80%" semantics
- Metric not aggregated → create a **CloudWatch math expression** (e.g., `AVG(CPU of ASG)` or `MAX over instances`) and alarm on the expression
- Missing data → set `treat-missing-data: notBreaching` + fill data `missing: FillData`
- Period too coarse → use 60s periods with 5 evaluation periods instead of 300s × 1

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "AppCPUHigh" \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --statistic Average \
  --dimensions Name=AutoScalingGroupName,Value=web-asg \
  --period 60 --evaluation-periods 5 \
  --threshold 80 --comparison-operator GreaterThanThreshold \
  --treat-missing-data notBreaching \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:oncall-alerts
```

## Architecture

```
    Alarm pipeline — where it silently breaks
    ─────────────────────────────────────────

    App/EC2 emits metric ──▶ CloudWatch (AWS/EC2 CPUUtilization)
                                  │
                                  ▼
                         (period + statistic per datapoint)
                                  │
                                  ▼
                        Alarm evaluation every <period>
                                  │
                     window = Period × EvaluationPeriods
                                  │
                            threshold breach?
                                  │
                     ┌────────────┴─────────────┐
                     │ NO / missing data (treat  │
                     │ as OK by default)         │
                     │        → state stays OK   │
                     └───────────────────────────┘
                                  │
                                  ▼
                          ELIF ALARM state
                                  │
                                  ▼
                           SNS topic (publish)
                                  │   subscription active & confirmed?
                                  ▼
                            Page on-call
```

The common silent breakages:
1. `TreatMissingData` default treats holes as "not breaching" → OK forever
2. Wrong statistic (Average vs. Maximum) hides 95% instance CPU
3. Unaggregated ASG metric — per-instance average == 95 but ASG-level average == 60
4. No `alarm-actions` attached → alarm flips but nothing publishes; SNS sub unconfirmed

## Investigation

1. **Print the full alarm config** — threshold, statistic, period, evaluation periods, dimensions, treat-missing-data, actions
2. **Query the raw metric with the SAME period+statistic** — compare to dashboard display (they can differ)
3. **Verify the metric is being emitted** — `get-metric-statistics` shows data or empty — empty = namespace/dimension/write problem
4. **Review alarm history** — OK ↔ INSUFFICIENT clues vs. INSUFFICIENT_DATA gaps
5. **Check the aggregation level** — single-instance metric isn't ASG-average; use math expression
6. **Test the SNS path** — publish a test message directly to confirm topic + subscription work
7. **Check IAM** — the app IAM role has `cloudwatch:PutMetricData` on nothing? Custom metrics from ECS/EKS need `PutMetricData`
8. **Verify period resolution** — standard vs high-res; high-res cannot use period > certain sizes; standard periods are 60/300
9. **Look at all instances** — one hot instance doesn't breach an ASG-average alarm; correlation with SLO may be with a different metric
10. **Check for alarm throttling / data gaps during scale** — autoscaling can produce gaps the trace shows as INSUFFICIENT

## Commands

```bash
# Full config
aws cloudwatch describe-alarms --alarm-names AppCPUHigh --output json

# Raw data as the alarm sees it
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 --metric-name CPUUtilization \
  --dimensions Name=AutoScalingGroupName,Value=web-asg \
  --statistics Average --period 300 \
  --start-time "$(date -u -d '25 min ago' +%Y-%m-%dT%H:%M:%SZ)" --end-time "$(date -u)" \
  --output table

# History
aws cloudwatch describe-alarm-history --alarm-name AppCPUHigh --output table

# Metric with no data?
# Emissions check via list-metrics for exact dimension
aws cloudwatch list-metrics --namespace AWS/EC2 --metric-name CPUUtilization \
  --dimensions Name=AutoScalingGroupName,Value=web-asg

# Test SNS
aws sns publish --topic-arn arn:aws:sns:us-east-1:...:oncall-alerts \
  --message "Test alarm delivery"

# Create a math-expression alarm for max of instances
aws cloudwatch put-metric-alarm \
  --alarm-name "AppCPUHigh-Max" \
  --metrics '[{"Id":"m1","MetricStat":{"Metric":{"Namespace":"AWS/EC2","MetricName":"CPUUtilization","Dimensions":[{"Name":"AutoScalingGroupName","Value":"web-asg"}]},"Period":60,"Stat":"Average"}}]' \
  --statistic Maximum --period 60 --evaluation-periods 5 \
  --threshold 80 --comparison-operator GreaterThanThreshold \
  --treat-missing-data notBreaching \
  --alarm-actions arn:aws:sns:us-east-1:...:oncall-alerts
```

## Root Cause

| Root Cause | Evidence | Fix |
|---|---|---|
| TreatMissingData = missing (default) | Metric has gaps; alarm stays OK despite data gaps | Set `notBreaching` + gap-fill |
| Wrong statistic (Average hides 95%) | ASG average 60 while single instance 95 | Alarm on `Maximum` or math expression |
| Unaggregated metric | Alarm on per-instance dim, not ASG+aggregation | Add math expression for aggregate |
| Wrong period | 300s period averages spike into the window | Use 60s periods / high-res |
| Threshold mismatch | Threshold 95 vs actual alert 80 | Correct threshold |
| No alarm action / dead SNS | Alarm flips but no message | Attach actions; confirm subscription |
| Metric never published | list-metrics shows no data | Fix instrumentation / IAM `PutMetricData` |
| Wrong namespace/dimensions | Alarm metrics empty | Fix names/dims in metric-filters + app |
| Multi-account metrics | Alarm in account A, metric from B | Add `crossAccount`/OAM; or centralize |
| Alarm disabled | State=DISABLED | `enable-alarm-actions` |

## Immediate Mitigation

1. **Confirm what the metric really shows; match the dashboard to it.** If one instance is at 95% and the alarm is ASG-avg, the alarm is technically correct — but it's the wrong signal.
   - Action: create and arm a **Maximum-based math expression** alarm that matches the intent ("any instance above 80%").
2. **Deliver the alert now** — publish a manual test through the SNS topic to confirm on-call receives it.
3. **If data gaps:** switch alarm to `treat-missing-data: notBreaching` and add `missing: FillData` — prevents OK-stuck.
4. **Reduce period + evaluation count** to hit the latency of the true signal (e.g., `--period 60 --evaluation-periods 5`) and test with a manual `set-alarm-state ALARM`.
5. **Temporarily add an alert on ALB 5xx error rate** which reflects user impact today while CPU alarm problems get fixed.

## Permanent Fix

1. **Standardize alarm-lint** — run a script/CI check validating each alarm has: correct statistic, explicit `TreatMissingData`, attached actions, active SNS subscription
2. **Load test with metrics validation** — every major release: run a spike test and verify the alarms actually fire within the SLO
3. **Use math expressions for aggregate intent** — `MAX(AVG of ASG instances)` or `AVG of instance-level p99`
4. **Add "alarm state drift" false-positive guard** — alert when any critical alarm stays OK while a `MetricAlarm` with high CPU exists (contradiction detection)
5. **Central alert catalog** in IaC (Terraform `aws_cloudwatch_metric_alarm`), alerts versioned and reviewed with SLOs
6. **Test page delivery monthly** — chaos drill that forces a state change and confirms delivery to correct escalation

## Monitoring

```yaml
# Alarm health monitoring (meta-monitoring!)
- alert: CriticalAlarmDisabled
  expr: aws_cloudwatch_alarm_state_status == "disabled" and priority == "critical"
  for: 10m
  severity: critical

- alert: AlarmInInsufficientDataLongerThan1h
  expr: aws_cloudwatch_alarm_state == "INSUFFICIENT_DATA" for > 60m
  severity: warning

# Alert on delayed/duplicated delivery
#   CloudWatch does not dedupe in real-time — watch every delivery to the topic
- alert: OnCallTopicDeliveryFailure
  expr: aws_sns_failure_total > 0
```
Additionally, use **Composite Alarms** to reduce noise (e.g., CPU high AND error rate high → page).

## Security

- Least-privilege IAM: app only `cloudwatch:PutMetricData` on its namespace, never `DeleteAlarms`
- Guard against **alarm deletion/disabling** by rogue IAM (guard via role boundary)
- SNS topics encrypted with KMS; subscriptions restricted to authorized peers
- Store alarm definitions in Git (IaC) to detect drift in console — CloudTrail shows `PutMetricAlarm` events
- Monitor alarm CRUD keys via CloudTrail alert: unexpected `DeleteAlarm` from a prod role is suspicious

## Production Considerations

- **Cost**: alarms are cheap, but *per-metric custom dashboards* get pricey — aggregate, then alarm
- **Reliability**: alarm should echo the user impact signal (latency/5xx/error_rate) — not tunnel vision on CPU alone
- **Operational**: on-call must be able to confirm delivery path in < 5 min; put alarm health in the dashboards
- **Compliance**: SLO/SLA monitoring often needs documented alarm definitions + evidence of delivery (CloudTrail log stream)
- **Multi-account**: use CloudWatch Observability Access Manager (OAM) to centralize alarm state across accounts — an alarm that is in another account won't fire without it

## Senior-Level Answer

"I read the alarm's actual config, not the dashboard. The three classic silent killers: the statistic (average hides a hot instance), TreatMissingData (default keeps the alarm OK on missing data), and period/evaluation windows that average the spike away. I pull `describe-alarms` and compare its metric (same namespace, dimensions, statistic, period) to `get-metric-statistics` — if the alarm reads structured ASG-average while the dashboard shows per-instance CPU, that's mismatched signal, not a broken alarm. Then I check alarm history for OK↔INSUFFICIENT flapping and verify the SNS topic accepts a manual publish. The fix is a math-expression alarm on the intended semantics ('any instance over 80% for 5m'), explicit treat-missing-data, and a delivery test. Long-term, alarms live in IaC and I enforce every critical alarm has an attached, subscription-active action plus a monthly delivery drill."

## Architect-Level Answer

"Alarm reliability is part of observability design. I treat each alarm as a contract: a metric spec, an aggregation rule, an evaluation window, an action, and a delivery target — versioned in IaC. The failure modes here justify three principles: (1) intent-over-syntax — build expressions that match what operators actually mean (e.g., MAX over ASG instances), not the default Average; (2) fail-open on gaps — every alarm sets TreatMissingData explicitly, and gap periods fan out to INSUFFICIENT + an 'alert on alert health' meta-alarm; (3) delivery testing — monthly synthetic alarm that forces ALARM state, asserts SNS publication, page received, and auto-recovers to OK. I'd also move critical alerts to composite alarms (CPU high AND error high) to cut noise. Architecturally, alarms are configuration like any other: in Git, drifted-detectable via CloudTrail, and their health itself monitored continuously."

## Follow-Up Questions

1. "What specifically does 'for 5 minutes' mean in an alarm? Explain Period, EvaluationPeriods, and how high-resolution metrics change these."
2. "Why might an alarm sit in INSUFFICIENT_DATA instead of OK — and how does TreatMissingData change what happens?"
3. "Design a composite alarm that pages only when both user-facing errors and latency breach — include the CloudWatch math expression syntax."
4. "How do you validate end-to-end that an alarm change is correct before relying on it in production?"
5. "An alarm fires but the on-call never got paged. Walk through the SNS-and-pager delivery path and the checks to find the break."