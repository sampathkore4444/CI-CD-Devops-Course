# 57. etcd Performance Degradation

## Scenario

Your Kubernetes cluster has been running fine for 6 months. Suddenly, in the last 2 weeks, API operations started getting slower. `kubectl get pods` takes 8 seconds instead of 0.5 seconds. Deployments are timing out. The cluster-autoscaler is making bad decisions because it can't read node status fast enough. Monitoring shows etcd WAL fsync duration has gone from 10ms to 3 seconds. The etcd disk usage is at 85%. You need to diagnose what's causing etcd degradation and fix it before the cluster becomes completely unmanageable.

## Interviewer Question

"etcd response time has degraded from milliseconds to 5+ seconds. Walk me through diagnosing etcd performance issues, the tools you'd use, and how to resolve them without impacting running workloads."

## What I Should Think About

- etcd is the backbone of Kubernetes — all cluster state lives there
- Performance issues manifest as slow API server responses
- Key metrics: WAL fsync duration, disk I/O, memory usage, DB size, leader elections
- etcd is extremely disk-sensitive — IOPS and latency matter more than throughput
- Compaction and defrag are critical maintenance tasks
- Network latency between etcd peers affects performance
- Never ignore leader changes — they indicate instability
- Scaling etcd horizontally isn't straightforward — vertical scaling and disk quality matter most

## Ideal Answer

**Step 1: Assess etcd health**

```bash
# Check etcd endpoint health and latency
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health

# Check detailed status
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status --write-out=table

# Check DB size
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status --write-out=json | jq '.[] | {dbSize: .Status.dbSize, dbSizeInUse: .Status.dbSizeInUse, leader: .Status.leader}'
```

**Step 2: Check key metrics**

```bash
# Check etcd metrics
curl -s http://localhost:2379/metrics | grep -E "etcd_disk_wal_fsync_duration_seconds|etcd_disk_backend_commit_duration_seconds|etcd_server_has_leader|etcd_server_leader_changes_seen_total"

# Check disk I/O on etcd disk
iostat -x 1 10
iotop -oP

# Check if etcd is on SSD
lsblk -d -o name,rota
# rota=0 means SSD, rota=1 means HDD
```

**Step 3: Identify root cause and fix**

1. **Disk I/O bottleneck** (most common):
   ```bash
   # Check if etcd is on slow disk
   df -h /var/lib/etcd
   lsblk
   # Move etcd to SSD or increase IOPS
   ```

2. **DB size too large** — too many secrets, configmaps, or events:
   ```bash
   # Check total number of objects
   kubectl get secrets --all-namespaces | wc -l
   kubectl get configmaps --all-namespaces | wc -l
   kubectl get events --all-namespaces | wc -l

   # Set up event TTL (default 1 hour)
   # In kube-apiserver flags:
   # --event-ttl=1h

   # Enable garbage collection for large objects
   ```

3. **Missing compaction/defrag**:
   ```bash
   # Check current revision
   ETCDCTL_API=3 etcdctl endpoint status --write-out=json | jq '.[0].Status.revision'

   # Perform compaction (keep last hour)
   CURRENT_REV=$(ETCDCTL_API=3 etcdctl endpoint status --write-out=json | jq '.[0].Status.revision')
   COMPACT_REV=$((CURRENT_REV - 3600000))  # ~1 hour
   ETCDCTL_API=3 etcdctl compact $COMPACT_REV

   # Perform defrag
   ETCDCTL_API=3 etcdctl defrag
   ```

4. **Too many watchers**:
   ```bash
   # Check watch count
   curl -s http://localhost:2379/metrics | grep etcd_debugging_mvcc_db_total_size_in_bytes
   ```

## Architecture

```
    ┌─────────────────────────────────────────────────────────┐
    │                    etcd Cluster (3 nodes)               │
    │                                                         │
    │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
    │  │  etcd Node 1 │  │  etcd Node 2 │  │  etcd Node 3 │  │
    │  │  (Leader)    │  │  (Follower)  │  │  (Follower)  │  │
    │  │              │  │              │  │              │  │
    │  │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │  │
    │  │ │WAL File  │ │  │ │WAL File  │ │  │ │WAL File  │ │  │
    │  │ │(Write    │ │  │ │(Replicate│ │  │ │(Replicate│ │  │
    │  │ │Ahead Log)│ │  │ │  from    │ │  │ │  from    │ │  │
    │  │ └──────────┘ │  │ │ Leader)  │ │  │ │ Leader)  │ │  │
    │  │ ┌──────────┐ │  │ └──────────┘ │  │ └──────────┘ │  │
    │  │ │Backend   │ │  │ ┌──────────┐ │  │ ┌──────────┐ │  │
    │  │ │(BoltDB)  │ │  │ │Backend   │ │  │ │Backend   │ │  │
    │  │ │          │ │  │ │(BoltDB)  │ │  │ │(BoltDB)  │ │  │
    │  │ └──────────┘ │  │ │          │ │  │ │          │ │  │
    │  └──────────────┘  │ └──────────┘ │  │ └──────────┘ │  │
    │                     └──────────────┘  └──────────────┘  │
    └───────────────────────────┬─────────────────────────────┘
                                │
                    ┌───────────▼───────────────┐
                    │     Kubernetes API Server  │
                    │     (reads/writes to etcd) │
                    └───────────────────────────┘

    Performance Bottleneck Points:
    1. WAL fsync → Disk I/O (SSD critical)
    2. Backend commit → Disk I/O
    3. Peer communication → Network latency
    4. DB size → Memory and compaction
```

## Investigation

1. **Check etcd health endpoint** — `endpoint health` gives immediate status
2. **Check etcd metrics** — WAL fsync, backend commit, leader changes
3. **Check disk performance** — `iostat`, `iotop`, disk type (SSD vs HDD)
4. **Check DB size** — too many keys/objects bloat the database
5. **Check compaction history** — is compaction running regularly?
6. **Check network latency** — RTT between etcd peers
7. **Check memory usage** — etcd needs RAM for caching
8. **Check CPU usage** — serialization/deserialization is CPU-bound
9. **Check for split-brain** — multiple leader elections indicate instability
10. **Check Kubernetes objects** — excessive secrets, events, or configmaps

## Commands

```bash
# Full etcd health check
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status --write-out=table

# Check DB size vs DB size in use
# dbSizeInUse / dbSize < 0.7 means fragmentation is high → defrag needed

# Check WAL sync latency
curl -s http://localhost:2379/metrics | grep etcd_disk_wal_fsync_duration_seconds_bucket

# Check backend commit latency
curl -s http://localhost:2379/metrics | grep etcd_disk_backend_commit_duration_seconds_bucket

# Count all etcd keys
ETCDCTL_API=3 etcdctl get / --prefix --keys-only | wc -l

# Check which prefixes have the most keys
ETCDCTL_API=3 etcdctl get / --prefix --keys-only | cut -d'/' -f1-3 | sort | uniq -c | sort -rn | head -20

# Check disk I/O
iostat -x 1 5

# Check memory
free -m

# Monitor in real-time
watch -n1 'ETCDCTL_API=3 etcdctl endpoint status --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key --write-out=table'
```

## Root Cause

| Root Cause | Symptom | Fix |
|---|---|---|
| Slow disk (HDD) | High WAL fsync, high I/O wait | Move to SSD, use provisioned IOPS |
| DB fragmentation | dbSizeInUse << dbSize | Run defrag during maintenance |
| Too many keys | High DB size, slow reads | Compact old revisions, clean up objects |
| Missing compaction | Revision grows indefinitely | Set up automated compaction |
| Memory pressure | High RSS, OOM kills | Increase memory limits |
| Network latency between peers | High RTT, slow replication | Ensure low-latency network between nodes |
| CPU bottleneck | High serialization time | Increase CPU allocation |
| Too many watchers | High memory usage | Reduce unnecessary watches |

## Immediate Mitigation

```bash
# 1. Compact old revisions (reduces DB size)
CURRENT_REV=$(ETCDCTL_API=3 etcdctl endpoint status --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  --write-out=json | jq '.[0].Status.revision')
COMPACT_REV=$((CURRENT_REV - 3600000))
ETCDCTL_API=3 etcdctl compact $COMPACT_REV

# 2. Defrag (reclaims space)
ETCDCTL_API=3 etcdctl defrag --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# 3. Clean up excess events
kubectl delete events --all-namespaces --field-selector reason=Pulling --timeout=30s

# 4. Increase API server timeouts temporarily
# In kube-apiserver: --request-timeout=60s (from default 60s)
```

## Permanent Fix

1. **Move etcd to dedicated SSD nodes** — don't share with other workloads
2. **Automate compaction** — set `--auto-compaction-retention=1h` in etcd
3. **Set up regular defrag** — run during maintenance windows via CronJob
4. **Monitor DB size** — alert when > 8GB (recommended max is 8GB)
5. **Clean up Kubernetes objects** — remove old secrets, configmaps, events
6. **Set resource limits** — etcd needs 2-8GB RAM depending on cluster size
7. **Network optimization** — ensure etcd nodes are in same AZ with low latency

## Monitoring

```yaml
# Prometheus alerts for etcd
- alert: EtcdHighDiskLatency
  expr: histogram_quantile(0.99, rate(etcd_disk_wal_fsync_duration_seconds_bucket[5m])) > 0.5
  for: 5m
  labels:
    severity: warning

- alert: EtcdDBSizeHigh
  expr: etcd_mvcc_db_total_size_in_bytes > 8589934592  # 8GB
  for: 5m
  labels:
    severity: critical

- alert: EtcdLeaderChanges
  expr: increase(etcd_server_leader_changes_seen_total[1h]) > 3
  for: 5m
  labels:
    severity: warning

- alert: EtcdNoLeader
  expr: etcd_server_has_leader == 0
  for: 1m
  labels:
    severity: critical
```

## Security

- etcd stores ALL cluster secrets — encrypt at rest with KMS provider
- Never expose etcd to external networks — keep on private network only
- Use mutual TLS between etcd peers and between API server and etcd
- Restrict etcd access to only API server — use firewall rules
- Regularly rotate etcd certificates
- Audit access to etcd data — it contains all cluster state including secrets

## Production Considerations

- **HA**: Run 3 or 5 etcd nodes (odd number for Raft quorum)
- **Disk**: Use dedicated SSD volumes with high IOPS (gp3/io2 on AWS)
- **Network**: Keep etcd in same AZ or use low-latency cross-AZ links
- **Backup**: Daily etcd snapshots to S3, test restore quarterly
- **Sizing**: 2 CPU, 8GB RAM minimum for small clusters; scale up for large
- **Maintenance**: Schedule compaction/defrag during low-traffic periods
- **Cost**: Dedicated etcd nodes add cost but are essential for reliability

## Senior-Level Answer

"etcd is the single source of truth for Kubernetes, so its performance directly impacts the entire cluster. The most common cause of etcd degradation is slow disk I/O — etcd requires fast SSDs with low latency. I'd check WAL fsync duration, backend commit latency, and DB size. If the disk is the bottleneck, I'd migrate to SSD. I'd also check if compaction and defrag are running regularly — uncleaned revisions accumulate and slow down reads. For immediate relief, I'd compact old revisions and defrag the database. Long-term, I'd set up automated compaction, move etcd to dedicated nodes with SSD, and monitor DB size and fsync latency."

## Architect-Level Answer

"etcd performance is a critical architectural concern for Kubernetes at scale. The key design decisions are: (1) dedicated etcd nodes separate from API server — this isolates resource contention, (2) SSD storage with provisioned IOPS — etcd is I/O bound, (3) odd-numbered clusters (3 or 5) for quorum resilience, (4) same-AZ deployment to minimize network RTT, (5) automated compaction and defrag as part of cluster maintenance, (6) regular backups with tested restore procedures. For very large clusters, consider etcd quota management — the 8GB default limit requires active cleanup of old objects. I'd also implement circuit-breaker patterns in the API server to prevent cascade failures when etcd is slow."

## Follow-Up Questions

1. "How does etcd's Raft consensus protocol handle leader election, and what happens to API requests during leader changes?"
2. "What's the difference between compaction and defrag in etcd, and when would you use each?"
3. "If etcd DB size reaches the 8GB quota, what happens to the cluster and how do you recover?"
4. "How would you perform a rolling etcd upgrade without downtime?"
5. "Design an etcd backup and restore strategy — what's your RPO and RTO?"
