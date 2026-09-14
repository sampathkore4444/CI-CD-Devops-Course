# 2. Server CPU at 100% - Application Unresponsive

## Scenario
It's 10:15 AM on a Wednesday. The support team receives a flood of tickets: "Application is extremely slow" and "Pages are timing out." You check the monitoring dashboard and see that prod-app-server-02 has CPU at 100% sustained for the last 12 minutes. The server runs multiple Java microservices via Docker Compose: a Spring Boot API, a Kafka consumer, a batch processing worker, and an Elasticsearch node. Load average has spiked to 28 (on a 16-core machine). The application was working fine until about 20 minutes ago, after a deployment of the batch processing worker service at 9:55 AM. You need to identify which process is consuming CPU and resolve it.

## Interviewer Question
"Users report your application is extremely slow. You check the server and find CPU usage is at 100% across all cores. Multiple Java microservices are running via Docker. Walk me through how you identify which process is consuming CPU, determine why, and resolve it without affecting the other services."

## What I Should Think About
- **Correlation with recent changes**: A deployment happened 20 minutes ago — is it the cause?
- **Per-core vs. overall**: Is it one thread spinning or all cores saturated?
- **Java-specific**: Thread dumps, GC storms, JIT compilation issues
- **Docker CPU limits**: Were CPU limits set on containers?
- **Type of CPU usage**: User time vs. system time vs. iowait
- **Is it actually CPU?**: Could be iowait masquerading as CPU usage
- **Safe kill vs. graceful degradation**: Can I restart just the offending container?

## Ideal Answer

**Phase 1: Confirm and Categorize**

First, confirm the CPU situation and differentiate between user CPU, system CPU, iowait, and steal time:

```bash
top -bn1 | head -20
mpstat -P ALL 1 3
```

If `iowait` is high, this is actually a disk I/O problem, not CPU. If `steal` time is high, it's a hypervisor/VM contention issue.

**Phase 2: Identify the Offending Process**

```bash
# Top CPU consumers
ps aux --sort=-%cpu | head -20

# Or more precisely, by thread
ps -eLo pid,tid,pcpu,comm --sort=-pcpu | head -20

# In Docker context
docker stats --no-stream
```

This will show which container is consuming the most CPU. In our scenario, it's likely the batch processing worker that was recently deployed.

**Phase 3: Drill into the Container**

```bash
# Get the container ID
docker ps | grep batch-worker

# Exec into it to see Java processes
docker exec -it <container-id> ps aux

# Get thread-level detail
docker exec -it <container-id> top -H -bn1
```

**Phase 4: Java-Specific Investigation**

Since these are Java services, I'd take a thread dump:

```bash
# Find the Java PID inside the container
docker exec <container-id> jps -l

# Take a thread dump
docker exec <container-id> jstack <java-pid> > /tmp/thread-dump-$(date +%s).txt

# Check GC activity
docker exec <container-id> jstat -gcutil <java-pid> 1000 5
```

If the CPU spike correlates with the recent deployment, it's likely an infinite loop, excessive garbage collection, or a thread pool misconfiguration in the batch worker.

**Phase 5: Immediate Mitigation**

If the batch worker is the culprit:
```bash
# Option A: Apply CPU limit immediately (if not set)
docker update --cpus="2" <container-name>

# Option B: Pause the container
docker pause <container-name>

# Option C: Restart just the offending container
docker restart <container-name>

# Option D: Rollback to previous version
docker-compose up -d --no-deps batch-worker
```

**Phase 6: Post-Mitigation Analysis**

After the immediate fix, analyze the thread dump and GC logs to understand the root cause — was it an infinite loop, a query that returned unbounded results, a GC death spiral, or a configuration error?

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│                  prod-app-server-02                       │
│                  16 CPU cores                            │
│                                                          │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────────┐  │
│  │ API GW   │ │ Kafka    │ │ Batch    │ │ Elastic    │  │
│  │ 5% CPU   │ │ 3% CPU   │ │ 89% CPU  │ │ 3% CPU     │  │
│  │ ✓ OK     │ │ ✓ OK     │ │ ✗ FAULTY │ │ ✓ OK       │  │
│  └──────────┘ └──────────┘ └─────┬────┘ └────────────┘  │
│                                  │                       │
│                    ┌─────────────▼──────────────┐        │
│                    │   Thread Dump Analysis     │        │
│                    │   - Infinite loop in       │        │
│                    │     processBatch()         │        │
│                    │   - Unbounded result set   │        │
│                    │   - No CPU limit set       │        │
│                    └────────────────────────────┘        │
└──────────────────────────────────────────────────────────┘
```

## Investigation

1. **Check overall CPU breakdown**: Run `mpstat -P ALL 1 5` to see if it's user CPU, system CPU, iowait, or steal time.
2. **Identify top processes**: Run `ps aux --sort=-%cpu | head -20` to find which process is consuming the most.
3. **Check Docker container CPU usage**: Run `docker stats --no-stream` to see per-container CPU consumption.
4. **Check recent deployments**: Correlate the CPU spike timeline with deployment timestamps using `docker inspect --format='{{.State.StartedAt}}' <container>`.
5. **Inspect thread-level CPU**: Run `ps -eLo pid,tid,pcpu,comm --sort=-pcpu | head -20` to find the specific thread.
6. **Take a Java thread dump**: Run `docker exec <container> jstack <pid>` to capture thread states.
7. **Check GC activity**: Run `docker exec <container> jstat -gcutil <pid> 1000 5` to see if GC is consuming CPU.
8. **Check Docker CPU limits**: Run `docker inspect --format='{{.HostConfig.NanoCpus}}' <container>` to verify if limits were set.

## Commands

```bash
# 1. Overall CPU stats
top -bn1
mpstat -P ALL 1 3

# 2. Top CPU consumers (system-wide)
ps aux --sort=-%cpu | head -20

# 3. Thread-level CPU consumers
ps -eLo pid,tid,pcpu,comm --sort=-pcpu | head -30

# 4. Docker container CPU usage
docker stats --no-stream

# 5. Apply CPU limit to a container without restart
docker update --cpus="2" <container-name>

# 6. Take a Java thread dump from a container
docker exec <container> jstack $(docker exec <container> jps -l | grep -v Jps | awk '{print $1}')

# 7. Check GC activity
docker exec <container> jstat -gcutil $(docker exec <container> jps -l | grep -v Jps | awk '{print $1}') 1000 10

# 8. Check for runaway child processes
pstree -p <pid>

# 9. Check CPU cgroup limits
cat /sys/fs/cgroup/cpu/docker/<container-id>/cpu.cfs_quota_us
cat /sys/fs/cgroup/cpu/docker/<container-id>/cpu.cfs_period_us

# 10. Safely restart just one container
docker-compose restart batch-worker

# 11. Rollback a container to previous image
docker-compose up -d --no-deps --force-recreate batch-worker

# 12. Check strace on a specific process (use sparingly in production)
strace -c -p <pid> -e trace=all
```

## Root Cause

| Root Cause | Detection | Resolution |
|---|---|---|
| Infinite loop in application code | Thread dump shows thread stuck in same code location | Fix the code, deploy patch, restart container |
| GC death spiral (too much memory pressure) | `jstat` shows GC runs consuming >30% of time | Tune heap size, fix memory leak, or allocate more memory |
| CPU-intensive query with no limit | Thread dump + application logs show specific query | Add query timeout, add WHERE clause limit, add CPU limit |
| Misconfigured thread pool | Thread dump shows hundreds of threads | Configure thread pool max size properly |
| Fork bomb or runaway child processes | `pstree` shows deep process tree | Kill the root process, implement `--pids-limit` on Docker |
| No CPU limits set on containers | `docker inspect` shows NanoCpus = 0 | Set CPU limits on all containers |

## Immediate Mitigation

1. **Apply CPU limit immediately**: `docker update --cpus="2" <container>` — this throttles the container without restarting it, allowing other services to recover.
2. **If the container is unresponsive**: `docker pause <container>` freezes it completely, freeing CPU for other services.
3. **Restart the offending container**: `docker restart <container>` — this is the quickest way to stop a runaway process.
4. **If the container restarts with the same issue**: Roll back to the previous image version.
5. **Communicate** that the batch processing service is being temporarily throttled/down while the issue is investigated.

## Permanent Fix

1. **Set CPU limits on all containers** in docker-compose.yml:
```yaml
deploy:
  resources:
    limits:
      cpus: '2.0'
      memory: 2G
```
2. **Implement proper health checks** so orchestrators can detect and restart unhealthy containers.
3. **Add application-level circuit breakers** for CPU-intensive operations.
4. **Implement deployment canarying** to catch issues before full rollout.
5. **Add CPU profiling** (async-profiler for Java) to periodically capture performance profiles.
6. **Tune Java GC settings** appropriately for the workload.
7. **Implement request/operation timeouts** in batch processing to prevent unbounded work.

## Monitoring

- **CPU usage per container**: Alert when any container exceeds 80% CPU for 5+ minutes.
- **Load average**: Alert when load average exceeds 1.5x the number of CPU cores.
- **GC pause time**: Alert when GC pause exceeds 500ms or GC CPU usage exceeds 20%.
- **Thread count**: Alert when thread count in a JVM exceeds expected baseline.
- **Application response time**: Alert when P99 latency exceeds SLA.
- **Deployment events**: Correlate CPU spikes with deployment events in monitoring.

## Security

- **CPU exhaustion as DoS**: A compromised container could intentionally consume all CPU. CPU limits act as a safeguard against this.
- **Resource starvation attacks**: Without limits, one container can starve all others on the same host.
- **strace in production**: Be cautious with `strace` — it can slow down the target process significantly. Use it briefly and sparingly.
- **Container escape risk**: If an attacker can exhaust CPU, they may be able to exploit timing-based vulnerabilities.

## Production Considerations

- **HA**: If this server is behind a load balancer, consider draining traffic before making changes.
- **Batch processing timing**: Ensure batch jobs run during low-traffic windows.
- **Right-sizing**: If the server consistently can't handle the workload, it may need to be resized.
- **Container orchestration**: Consider moving to Kubernetes where resource limits and auto-scaling are native features.
- **Deployment strategy**: Implement blue-green or canary deployments to minimize impact of bad releases.
- **Cost**: Over-provisioning CPU to handle spikes has cost implications. Balance between performance and cost.

## Senior-Level Answer

"I'd first verify whether it's truly CPU or potentially iowait masquerading as CPU usage using `mpstat`. Then I'd use `docker stats` and `ps aux --sort=-%cpu` to identify the offending container and process. Given the deployment 20 minutes ago, I'd correlate the spike with that change. I'd immediately apply a CPU limit using `docker update --cpus='2'` to throttle the offending container without affecting others. For a Java service, I'd capture a thread dump to identify the specific code path. After mitigating, I'd rollback the deployment and investigate the thread dump to find the root cause — likely an infinite loop or unbounded query in the batch worker."

## Architect-Level Answer

"This incident highlights the need for a defense-in-depth resource management strategy. At the infrastructure level, every container should have CPU and memory limits enforced via orchestration — not as an afterthought. At the application level, we need circuit breakers, timeouts, and rate limiting for CPU-intensive operations. At the deployment level, we need canary deployments with automatic rollback triggered by CPU/latency thresholds. At the observability level, we need per-container CPU metrics with anomaly detection, not just static thresholds. I'd also recommend implementing continuous profiling (e.g., continuous profiling with Pyroscope or Parca) so we have historical CPU profiles to compare against during incidents. The batch processing workload should ideally run on a separate node pool or have guaranteed CPU limits to never impact the latency-sensitive API services."

## Follow-Up Questions

1. "What's the difference between `docker update --cpus` and setting CPU limits in docker-compose.yml? When would you use each?"
2. "How would you handle this situation if all containers were at 100% CPU simultaneously?"
3. "Explain the difference between CPU limits and CPU reservations. How does each affect scheduling?"
4. "If the high CPU was caused by a Java GC storm, how would you tune the JVM to prevent it?"
5. "How would you implement automated rollback based on CPU metrics in a CI/CD pipeline?"
