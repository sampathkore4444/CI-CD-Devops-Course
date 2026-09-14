# 3. Server Running Out of Memory - OOM Killer Active

## Scenario
At 3:22 AM, the monitoring system sends a critical alert: "Memory usage critical on prod-db-replica-01: 98.2%." By the time you respond at 3:28 AM, it's at 100%. The server is a dedicated database replica running PostgreSQL 15, with a Redis cache instance and a log shipping agent. You start seeing "Out of memory: Killed process" messages in `dmesg`. The Redis process was killed, then the log shipper. PostgreSQL is still alive but barely. The primary database is unaffected, but if PostgreSQL gets killed, replication will break and you'll have a split-brain risk. The application is still running but read performance is degrading because the cache layer is gone.

## Interviewer Question
"The Linux server's memory usage has reached 100% and the OOM killer has started terminating processes. Critical application containers are being killed. Walk me through how you investigate the root cause, stop the bleeding, and resolve this while maintaining data integrity."

## What I Should Think About
- **OOM killer behavior**: It kills the process with the highest `oom_score_adj` or the largest memory consumer
- **Process criticality order**: PostgreSQL > Redis > log shipper — which ones are still alive?
- **Memory vs. swap**: Is swap configured? How much is in use?
- **Application-level caching**: Is Redis using too much memory? Is it a memory leak?
- **PostgreSQL memory**: Shared buffers, work_mem, effective_cache_size
- **Container memory limits**: Were they set? Were they appropriate?
- **Can I free memory without killing processes?**: Drop caches, reduce buffer pools
- **Data integrity**: If PostgreSQL dies mid-transaction, recovery is needed

## Ideal Answer

**Phase 1: Immediate Assessment (First 2 minutes)**

Check what's still alive and what the OOM killer has already killed:

```bash
dmesg | grep -i "oom\|out of memory\|killed process" | tail -20
free -h
ps aux --sort=-%mem | head -20
```

This tells me which processes were killed and what's still consuming memory.

**Phase 2: Stop the Bleeding**

If PostgreSQL is still running, protect it immediately:

```bash
# Lower OOM score for PostgreSQL to make it less likely to be killed
echo -1000 > /proc/$(pgrep postgres)/oom_score_adj

# Free up kernel caches (temporary relief)
echo 1 > /proc/sys/vm/drop_caches  # Free page cache
echo 2 > /proc/sys/vm/drop_caches  # Free dentries and inodes
echo 3 > /proc/sys/vm/drop_caches  # Free all

# Check and potentially increase swap
swapon --show
```

**Phase 3: Identify the Memory Consumer**

```bash
# Top memory consumers
ps aux --sort=-%mem | head -20

# Per-process memory detail
for pid in $(ps -eo pid --sort=-%mem | head -10 | tail -9); do
  echo "PID: $pid RSS: $(cat /proc/$pid/status 2>/dev/null | grep VmRSS) CMD: $(cat /proc/$pid/cmdline 2>/dev/null | tr '\0' ' ')"
done

# Check if Redis was using too much memory
redis-cli info memory 2>/dev/null

# Check PostgreSQL memory settings
psql -c "SHOW shared_buffers; SHOW work_mem; SHOW effective_cache_size;"
```

**Phase 4: Container-Specific Investigation**

```bash
# Check container memory limits and usage
docker stats --no-stream

# Check which containers were killed
docker ps -a --filter "status=exited" --format "{{.Names}}\t{{.Status}}"

# Check OOM-killed containers
docker inspect <container> | grep -i oom
```

**Phase 5: Long-term Fix**

The fix depends on the root cause:
- Redis memory leak: Fix the leak, set `maxmemory` limit
- PostgreSQL memory tuning: Adjust `shared_buffers`, `work_mem`
- Application memory leak: Profile and fix the leak
- Insufficient RAM: Upgrade the server

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│              prod-db-replica-01                          │
│              Total RAM: 32GB                            │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │ PostgreSQL 15                                     │   │
│  │ shared_buffers: 8GB (correct)                    │   │
│  │ work_mem: 256MB × 100 connections = 25GB ← ISSUE │   │
│  │ Status: Still alive, oom_score_adj = -1000       │   │
│  └──────────────────────────────────────────────────┘   │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │ Redis Cache                                       │   │
│  │ maxmemory: NOT SET ← ISSUE                       │   │
│  │ Used: 12GB and growing (memory leak)             │   │
│  │ Status: KILLED by OOM killer                     │   │
│  └──────────────────────────────────────────────────┘   │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │ Log Shipper                                       │   │
│  │ Status: KILLED by OOM killer                     │   │
│  └──────────────────────────────────────────────────┘   │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │ Swap: 8GB configured, 7.2GB used (90%)           │   │
│  │ Page cache: Minimal (thrashing)                  │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

## Investigation

1. **Check dmesg for OOM events**: Run `dmesg | grep -i "oom\|killed process"` to see what the OOM killer terminated and why.
2. **Check current memory state**: Run `free -h` to see total, used, free, and swap usage.
3. **Identify top memory consumers**: Run `ps aux --sort=-%mem | head -20` to find what's eating memory.
4. **Check for memory leaks**: Run `smem -t -k -s rss -r` for accurate per-process memory (includes shared memory properly).
5. **Check PostgreSQL memory settings**: Query `SHOW shared_buffers; SHOW work_mem; SHOW max_connections;` — `work_mem × max_connections` can easily exceed physical RAM.
6. **Check Redis memory**: Run `redis-cli info memory` to see `used_memory` and `maxmemory` settings.
7. **Check container memory limits**: Run `docker stats --no-stream` and `docker inspect` for memory constraints.
8. **Check for huge pages or shared memory segments**: Run `ipcs -m` and check `/proc/meminfo` for HugePages.

## Commands

```bash
# 1. Check OOM killer history
dmesg | grep -i "oom\|killed process" | tail -30

# 2. Current memory state
free -h
cat /proc/meminfo | grep -E "MemTotal|MemFree|MemAvailable|Buffers|Cached|SwapTotal|SwapFree|Committed_AS"

# 3. Top memory consumers
ps aux --sort=-%mem | head -20

# 4. Accurate memory accounting (includes shared libs)
smem -t -k -s rss -r | head -20

# 5. Check PostgreSQL memory configuration
sudo -u postgres psql -c "SHOW shared_buffers;"
sudo -u postgres psql -c "SHOW work_mem;"
sudo -u postgres psql -c "SHOW effective_cache_size;"
sudo -u postgres psql -c "SHOW max_connections;"

# 6. Check PostgreSQL memory usage per connection
sudo -u postgres psql -c "SELECT pid, usename, state, backend_type, 
  pg_size_of_relation(0) as temp_used 
  FROM pg_stat_activity;"

# 7. Check Redis memory
redis-cli info memory | grep -E "used_memory_human|maxmemory_human|maxmemory_policy"

# 8. Protect PostgreSQL from OOM killer
echo -1000 > /proc/$(pgrep -o postgres)/oom_score_adj

# 9. Free kernel caches (temporary)
sync && echo 3 > /proc/sys/vm/drop_caches

# 10. Check swap usage per process
for pid in $(ls /proc | grep -E '^[0-9]+$' | head -50); do
  swap=$(awk '/VmSwap/{print $2}' /proc/$pid/status 2>/dev/null)
  if [ -n "$swap" ] && [ "$swap" -gt 1000 ] 2>/dev/null; then
    cmd=$(cat /proc/$pid/cmdline 2>/dev/null | tr '\0' ' ' | head -c 80)
    echo "PID: $pid Swap: ${swap}kB CMD: $cmd"
  fi
done | sort -t: -k3 -rn | head -10

# 11. Set memory limit on a container
docker update --memory=4g --memory-swap=4g <container>

# 12. Add swap if none exists
fallocate -l 8G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
```

## Root Cause

| Root Cause | Detection | Resolution |
|---|---|---|
| `work_mem` too high × many connections | `SHOW work_mem` × `max_connections` exceeds RAM | Reduce `work_mem` to 16-64MB, use connection pooling |
| Redis maxmemory not configured | `redis-cli info memory` shows no maxmemory | Set `maxmemory` and eviction policy in redis.conf |
| Memory leak in application | Memory grows linearly over time without bound | Profile with `valgrind` or language-specific tools |
| Shared memory segments not cleaned up | `ipcs -m` shows large orphaned segments | `ipcrm -m <shmid>` to remove, fix the application |
| HugePages misconfigured | `grep Huge /proc/meminfo` shows large allocation | Tune or disable HugePages based on workload |
| Too many processes/connections | `ps aux | wc -l` shows excessive process count | Use connection pooling, reduce max connections |

## Immediate Mitigation

1. **Protect critical processes**: Set `oom_score_adj` to -1000 for PostgreSQL: `echo -1000 > /proc/<pgpid>/oom_score_adj`.
2. **Free kernel caches**: `sync && echo 3 > /proc/sys/vm/drop_caches` — this buys time but isn't a fix.
3. **Restart killed services**: Restart Redis and the log shipper: `systemctl restart redis`.
4. **Kill non-essential processes**: Find and kill any non-critical processes consuming memory.
5. **Reduce PostgreSQL connections**: Kill idle connections: `SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE state = 'idle' AND query_start < now() - interval '10 minutes';`
6. **Add emergency swap**: If no swap exists, create an 8GB swap file immediately.
7. **Set Redis maxmemory**: `redis-cli CONFIG SET maxmemory 4gb` and `redis-cli CONFIG SET maxmemory-policy allkeys-lru`.

## Permanent Fix

1. **Tune PostgreSQL memory**:
```sql
ALTER SYSTEM SET work_mem = '32MB';
ALTER SYSTEM SET shared_buffers = '8GB';
ALTER SYSTEM SET effective_cache_size = '24GB';
ALTER SYSTEM SET max_connections = 100;
-- Use PgBouncer for connection pooling
```
2. **Configure Redis maxmemory** in `redis.conf`:
```
maxmemory 4gb
maxmemory-policy allkeys-lru
```
3. **Set Docker memory limits** on all containers.
4. **Add swap** (2-4GB) as a safety net.
5. **Tune swappiness**: `echo 10 > /proc/sys/vm/swappiness`.
6. **Implement memory monitoring** with alerts at 80% and 90%.
7. **Profile the application** to find and fix any memory leaks.

## Monitoring

- **Memory usage**: Alert at 80% (warning) and 90% (critical).
- **Swap usage**: Alert when swap usage exceeds 50%.
- **OOM events**: Monitor `dmesg` for OOM killer messages and alert immediately.
- **PostgreSQL connections**: Alert when active connections exceed 80% of max.
- **Redis memory**: Alert when Redis memory exceeds 80% of maxmemory.
- **Process count**: Alert when process count deviates significantly from baseline.
- **Per-process memory growth**: Track memory growth rate of critical processes.

## Security

- **OOM killer manipulation**: A compromised process could set its own `oom_score_adj` to -1000 to avoid being killed, forcing the OOM killer to target critical processes. Use kernel-level restrictions.
- **Memory-based side-channel attacks**: Shared memory and cache timing can be exploited. Consider disabling Hyper-Threading for sensitive workloads.
- **容器逃逸**: Memory exhaustion can potentially be exploited for container escapes. Ensure proper resource limits.
- **Sensitive data in swap**: If swap is used, sensitive data (database page cache, encryption keys) may be written to disk. Consider encrypted swap.

## Production Considerations

- **HA**: If this is a database replica, ensure the primary is unaffected. Check replication lag after recovery.
- **Data integrity**: If PostgreSQL was OOM-killed, verify data integrity with `pg_resetwal` if needed.
- **Connection pooling**: Implement PgBouncer or Pgpool-II to manage connection count.
- **Right-sizing**: This server may need more RAM. Benchmark the workload and right-size the instance.
- **Container orchestration**: Kubernetes has native memory limits and OOM handling. Consider migration.
- **Cost**: More RAM costs more. Balance between right-sizing and over-provisioning.

## Senior-Level Answer

"I'd first check `dmesg` for OOM killer events and `free -h` for current memory state. I'd protect PostgreSQL by setting its `oom_score_adj` to -1000 and free kernel caches with `echo 3 > /proc/sys/vm/drop_caches` to buy time. Then I'd identify the memory consumer — in this case likely PostgreSQL's `work_mem` multiplied across connections, or Redis without a `maxmemory` limit. I'd restart the killed services, set Redis `maxmemory` to 4GB with LRU eviction, reduce PostgreSQL `work_mem` to 32MB, and implement PgBouncer for connection pooling. Long-term, I'd tune memory settings based on the actual workload profile and implement proper monitoring with graduated alerts."

## Architect-Level Answer

"This incident is a symptom of three architectural problems: lack of resource isolation, absence of capacity planning, and missing defensive configurations. Every service should have explicit memory limits — not just at the container level but at the application level (PostgreSQL `work_mem`, Redis `maxmemory`). The architecture should embrace resource budgeting: each service gets a defined memory budget and enforces it. I'd implement a service mesh or orchestrator (Kubernetes) that provides native resource isolation and OOM handling. For databases specifically, I'd separate the connection management layer (PgBouncer) from the database itself. I'd also implement a capacity planning process that includes regular load testing, memory profiling, and growth projections. The replica architecture should be re-evaluated — if it's serving reads, consider read replicas on separate instances to distribute memory pressure. Finally, implement chaos engineering practices to regularly test OOM scenarios in staging so the team is prepared."

## Follow-Up Questions

1. "How does the Linux OOM killer decide which process to kill? What is `oom_score` and how can you influence it?"
2. "Explain the difference between `work_mem`, `shared_buffers`, and `effective_cache_size` in PostgreSQL. How do they interact with OS memory?"
3. "What's the difference between a memory leak and memory fragmentation? How would you diagnose each?"
4. "How would you handle this situation if the OOM-killed process was the primary database, not a replica?"
5. "Design a memory management strategy for a microservices architecture running 50 services on a Kubernetes cluster."
