# 63. RDS Failover Causing Application Outage

## Scenario

It's 3:18 PM on a Tuesday. The monitoring page fires: "RDS Multi-AZ failover detected — database `payments-prod` failed over from us-east-1a to us-east-1b". Immediately after, the application error rate spikes. Customers see 504 timeouts for roughly 3 minutes. The error logs show "Connection refused" and "MySQL server has gone away" repeatedly. Investigation reveals the connection string in the app config points to the **reader endpoint** (`readonly.payments-prod.cluster-ro.us-east-1.rds.amazonaws.com`) instead of the **writer endpoint**. The reader endpoint returned a DNS error because at that exact moment there was no read replica available. Your job: fix the outage, fix the config, and make the app resilient to future failovers.

## Interviewer Question

"A Multi-AZ RDS failover happened automatically, yet the application had a 3-minute outage and errors. The connection string points at the reader endpoint instead of the writer. How do you fix this, and what should be in place so future failovers are transparent to the application?"

## What I Should Think About

- Multi-AZ failover should NOT require the app to change connection strings — that's the whole point
- Writer vs. reader endpoints: writer stays the same DNS name across failover; reader points to replicas
- App connection pooling breaks during failover — connections in pool die, need retry + reconnect
- idempotent retry logic is essential — failover windows are exactly when + partial commits happen
- Read timeouts, connection timeouts, "gone away" errors are normal transient symptoms to handle
- App-level: connection pool sizing, connection validation, auto-reconnect, read-write splitting
- Infrastructure: RDS Multi-AZ (Primary + standby), RDS Proxy for connection reuse, multi-region if needed
- Fix order: restore app NOW -> correct config -> harden retry logic -> test failover regularly
- Never aim to "prevent" failovers (Maintenance, patching legitimately cause them) — aim for zero app-visible impact

## Ideal Answer

**Step 1 — Immediately restore the app connection**

1. Check the running config: which endpoint is the app using for writes?
   ```
   mysql://payments:****@readonly.payments-prod.cluster-ro....:3306/paymentsdb
   ```
   That's the problem — the app writes to the **reader** endpoint.
2. Correct the connection string to the **writer** endpoint:
   ```
   mysql://payments:****@payments-prod.cluster-ro....:3306/paymentsdb
   ```
   Wait — check carefully: `.cluster-` (writer) vs `.cluster-ro-` (reader).

   Correct writer endpoint format: `payments-prod.cluster-xxx.us-east-1.rds.amazonaws.com`
   Reader endpoint format: `payments-prod.cluster-ro-xxx.us-east-1.rds.amazonaws.com`

3. Redeploy config + restart app within the deployment pipeline (rolling, zero-downtime if possible).

**Step 2 — Verify the failover actually recovered**

```bash
# Check the RDS instance
aws rds describe-db-instances --db-instance-identifier payments-prod \
  --query 'DBInstances[0].{Endpoint:Endpoint.Address,MultiAZ:MultiAZ,Status:DBInstanceStatus,AvailabilityZone:AvailabilityZone}'

# Check for the standby
aws rds describe-db-instances --db-instance-identifier payments-prod \
  --query 'DBInstances[0].[MultiAZ,SecondaryAvailabilityZone]'
```

**Step 3 — Harden the application to survive failover**

1. Use the correct writer/reader endpoints in config (or better, RDS Proxy)
2. Add connection validation + retry logic:
   - JDBC/MySQL connector with `autoReconnect=true`
   - Configure connection pool with `validate-on-borrow` and failover detection
3. Add bounded retries in the app service with exponential backoff + jitter
4. Set connection timeouts below your app's SLO
5. Test the full failover path in staging with a controlled failure

**Step 4 — Plan RDS Proxy to mask failover**

Create RDS Proxy — the app connects to the proxy endpoint, which survives failover transparently:
```bash
aws rds create-db-proxy \
  --db-proxy-name payments-proxy \
  --engine-family MYSQL \
  --auth file://proxy-auth.json \
  --role-arn arn:aws:iam::xxx:role/rds-proxy-role \
  --vpc-subnet-ids subnet-1a subnet-1b \
  --require-tls
```

The proxy keeps idle connections warm during failover, so the app's connections stay valid.

## Architecture

```
    BEFORE (misconfigured):
    ┌─────────┐      write       ┌──────────────────────────────┐
    │   App   │ ───────────────▶ │        READER endpoint       │
    │ (ECS)   │                  │  (DNS: *.cluster-ro-*)       │
    └─────────┘                  └──────────────┬───────────────┘
                                                │  redirects to...
                                               ▼
    ┌──────────────────────────────────────────────────────────┐
    │   RDS PRIMARY (writer)   │   STANDBY (read-only standby)  │
    │   us-east-1a             │   us-east-1b                   │
    └──────────────────────────────────────────────────────────┘
        ↑ AT FAILOVER: reader endpoint had no replica → DNS error → 3-min outage

    AFTER (correct):
    ┌─────────┐      ┌────────────────────────────────────────────┐
    │   App   │─────▶│  RDS Proxy (payments-proxy.rds.amazonaws.com) │
    │ (ECS)   │      │  survives failover, reuses warm connections │
    └─────────┘      └──────────────┬─────────────────────────────┘
                                    │
                    ┌───────────────▼────────────────────────────┐
                    │  ▶ WRITER endpoint (cluster-*) / READER    │
                    │    RDS Multi-AZ: Primary + Standby         │
                    └────────────────────────────────────────────┘

    Multi-AZ failover behavior:
    - DNS for the WRITER endpoint automatically points to the new primary
    - Connections to the old primary drop once (normal)
    - A strong app + proxy retries and masks that moment
```

## Investigation

1. **Check the running connection string** in app config / env / Secrets Manager
2. **Check for a reader vs. writer endpoint** — `cluster-ro-` vs `cluster-`
3. **Determine if the app is doing writes through the reader endpoint** (it should be read-only)
4. **Check the failover event**:
   ```bash
   aws rds describe-events --source-type db-instance --source-identifier payments-prod \
     --start-time "$(date -u -d '6 hours ago' +%Y-%m-%dT%H:%M:%SZ)"
   ```
5. **Check error stack** — look for the exact connect error and the DNS name attempted
6. **Check read replicas** — at failover time, was there at least one reader replica?
7. **Check app timeout/retry behavior** — no retries, small timeout = immediate 5xx
8. **Check connection pool settings** — pool may hold stale IP in the DNS cache for the old primary
9. **Check for a similar failure in the past** — is this a known pattern?
10. **Check if a failover test ran** — was a manual failover triggered recently?

## Commands

```bash
# Verify instance + endpoints
aws rds describe-db-instances --db-instance-identifier payments-prod \
  --query 'DBInstances[0].{Endpoint:.Endpoint.Address, Reader:.ReaderEndpointAddress, Cluster:.DBClusterIdentifier, MultiAZ:.MultiAZ, AZ:.AvailabilityZone, Secondary:.SecondaryAvailabilityZone}'

# Show recent events around failover
aws rds describe-events --source-type db-instance --source-identifier payments-prod \
  --start-time "$(date -u -d '6 hours ago' +%Y-%m-%dT%H:%M:%SZ)" --output table

# Check the config stored in Secrets Manager (mask password)
aws secretsmanager get-secret-value --secret-id payments/db

# Test endpoints
nslookup payments-prod.cluster-xxx.us-east-1.rds.amazonaws.com
nslookup payments-prod.cluster-ro-xxx.us-east-1.rds.amazonaws.com

# Check RDS proxy status
aws rds describe-db-proxies --db-proxy-name payments-proxy \
  --query 'DBProxies[0].{Engine:.Engine,Status:.Status,Writer:.Writer}

# Simulate a manual failover to test resiliency (maintenance window)
aws rds failover-db-cluster --db-cluster-identifier payments-prod \
  --target-db-instance-identifier payments-prod-eu-west-1b
```

## Root Cause

| Root Cause | Evidence | Fix |
|---|---|---|
| App configured with READER endpoint for writes | read-only endpoint in connection string | Switch to writer endpoint or RDS Proxy |
| No connection pool warming | Connection errors at failover moment | Add RDS Proxy + app pool validation |
| No retry logic in app | Immediate 504 on transient disconnect | Exponential backoff + jitter retry in app |
| Reader had no replica at failover | DNS `cluster-ro-*` returned no address | Ensure ≥1 replica, or use proxy/instance endpoint |
| Connection DNS caching | App cached the old primary IP | Configure short TTL / use proxy endpoint |
| Pool exhausted by failover storm | Connection pool too small for reconnect burst | Size pool to handle hot reconnect |
| Wrong subnet/SG for new primary | DNS resolves to new IP that's blocked | Fix SG/routing, use proxy in correct VPC |

## Immediate Mitigation

1. **Change the app connection string to the WRITER endpoint now** (or to an instance endpoint), deploy as hotfix with fast rollback:
   ```bash
   aws secretsmanager update-secret --secret-id payments/db \
     --secret-string '{"host":"payments-prod.cluster-xxx.us-east-1.rds.amazonaws.com",...}'
   ```
2. **Restart the ECS service** to pick up the new config (rolling to avoid total outage).
3. **Restart connection pool** if needed: scale down/up the service, or drain connections.
4. **Create an RDS Proxy** if the failover will recur — it extends availability:
   ```bash
   aws rds create-db-proxy --db-proxy-name payments-proxy ... # then point app at proxy
   ```
5. **Ensure a read replica exists** so the reader endpoint works while the primary is transitioning (if the app still references reader for reads).

## Permanent Fix

1. **Use RDS Proxy** everywhere — it survives failover, reuses warm connections:
   - App connects to proxy endpoint, pool stays warm
   - Proxy automatically redirects to new primary after failover
2. **Standardize connection strings** in IaC (Terraform) referencing writer endpoint for writes, reader for reads
3. **Add app-side retry logic** — any transient DB error triggers retry with backoff, idempotency keys for writes
4. **Read/write splitting** — write path → writer, read path → reader + replicas
5. **Set timeouts and pool sizes deliberately** — connectionTimeout below service SLO, pooling sized to restore failed connections in one failover window
6. **Automated failover drill monthly** — run `aws rds failover-db-cluster` in staging, then verify zero app impact
7. **Database connection monitoring** — track connections, wait events, and error rate in dashboards

## Monitoring

```yaml
# CloudWatch alarms
- alert: RDSFailoverDetected
  expr: AWS/RDS DBInstanceStatus == 'failing-over'
  for: 1m
  severity: critical (war-room + autocreate incident)

- alert: ConnectionsExceedThreshold
  expr: AWS/RDS DatabaseConnections > 90% of max
  for: 5m
  severity: warning  # pre-cursor to pool exhaustion during failover

- alert: App5xxDBErrors
  expr: sum(rate(http_requests{status=~"5[0-9][0-9]",db="payments"}[2m])) > 10
  for: 2m
  severity: critical
```

ECM alert: metric `RDSMetricBasedAlarmType=database.getFailover` using DB instance metrics (event-driven + rate-based).

## Security

- Rotate DB credentials frequently; store in Secrets Manager — never in env vars
- Restrict DB SG to app tier's SG only; no DB port open to the internet
- Use SSL/TLS connections to RDS (`require-tls` on proxy)
- Run `aws rds describe-db-security-groups` audit; verify no 0.0.0.0/0
- **RDS Proxy + Secrets Manager**: proxy rotates secrets without app restart
- Least privilege for DB users — separate read/write DB roles
- Ensure failover happens in same VPC/security boundary (proxy in app VPC)

## Production Considerations

- **HA**: Multi-AZ is mandatory for production DBs; consider multi-region replica for DR
- **Cost**: RDS Proxy adds cost (~15% of instance cost); multi-AZ doubles compute. Worth it for availability
- **Reliability**: failover is a feature, not a bug — patching/maintenance and rack failures trigger it, so the app MUST survive it
- **Scalability**: add read replicas for read-heavy load; use reader endpoint aware app for reads
- **Operational**: monthly failover drills with zero-downtime application alb checks
- **Compliance**: Multi-AZ + automatic backups with PITR satisfy many audit requirements (SOC2/PCI)
- **Data integrity**: enable Multi-AZ regardless — synchronous replication avoids data loss during failover

## Senior-Level Answer

"A failover shouldn't break a well-built app. The failure here was config: the app wrote through the reader endpoint (`cluster-ro-`), which had no replica at failover time. I'd fix the connection string to the writer endpoint and ensure retry+reconnect behavior in the pool. Then I'd add RDS Proxy so future failovers reuse warm connections transparently. I'd also confirm Multi-AZ plus at least one read replica, size the connection pool to survive reopen bursts, and run a monthly manual failover test in staging with zero-visible-impact as the acceptance criteria. The permanent architecture is writer endpoint for writes, reader for reads, proxy in front, retries in the app, and drills on the calendar."

## Architect-Level Answer

"Availability is designed, not configured. RDS Multi-AZ gives synchronous failover of the storage layer, but the application layer must cooperate: (1) connection management via RDS Proxy to surface a stable endpoint and pool-warming, (2) retry-with-backoff and idempotency in the data-access layer, (3) explicit read/write routing with separate writer and reader endpoints, (4) monthly failover drills with app-visible metrics verified to be green, (5) secrets in Secrets Manager with rotation transparent to the app. At the architecture level, I treat failover as an always-possible event: automatic detection, warm connections through the proxy, and a runbook-free manual escape hatch. The proxy removes the biggest variable — connection churn at the moment of failover — and is the difference between a blip and an outage."

## Follow-Up Questions

1. "Walk through exactly what happens to in-flight transactions during a Multi-AZ failover fight. How should the application handle the possibility of duplicate/partial writes?"
2. "What are the exact differences between a writer endpoint, reader endpoint, and instance endpoint in RDS/Aurora? When would you use each?"
3. "How does RDS Proxy improve failover behavior, and what are its connection-pinning limitations?"
4. "A developer argues that disabling auto-failover would prevent outages. Respond with why that's wrong and what the correct protection is."
5. "How would you verify an application's resilience to failover: design the load test, the checkpoints, and the pass criteria."