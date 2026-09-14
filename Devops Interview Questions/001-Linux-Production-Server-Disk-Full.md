# 1. Production Server 100% Disk Utilization

## Scenario
It's 2:47 AM on a Tuesday. PagerDuty fires an alert: "Disk usage critical on prod-app-server-03 (98.7%)." By the time you open your laptop, the alert has escalated to 100%. The server hosts a Dockerized microservices application running 12 containers — an API gateway, payment processing service, user authentication, and supporting services. Developers in the #incidents Slack channel are reporting 503 errors from the load balancer. The application logs show "No space left on device" errors. You need to free space immediately and identify the root cause — all without rebooting or causing additional downtime.

## Interviewer Question
"The production server hosting your Dockerized applications has hit 100% disk utilization. Applications are failing with 'No space left on device' errors. Walk me through your step-by-step approach to diagnose the root cause, recover the service, and implement measures to prevent recurrence."

## What I Should Think About
- **Triage first**: Is the service down or degraded? What's the blast radius?
- **Quick wins**: Can I free space without restarting anything?
- **Identify the culprit**: Was it a sudden spike or gradual growth?
- **Docker-specific concerns**: Docker logs, image layers, overlay2 filesystem, container writable layers
- **Application impact**: Are databases corrupted? Are writes partially committed?
- **Prevention**: logrotate, monitoring, alerts, disk quotas
- **Communication**: Keep stakeholders informed during the incident
- **Verification**: Confirm the fix actually works and isn't just masking the problem

## Ideal Answer

**Phase 1: Immediate Triage (First 5 minutes)**

First, I'd establish the scope of the impact. I'd check if the application is fully down or partially degraded by hitting the load balancer VIP or checking the health endpoints. I'd communicate in the incident channel that I'm investigating and provide an ETA.

Then I'd SSH into the server and get a quick overview of disk usage:

```bash
df -h
df -i  # Check inode usage too
du -sh /* 2>/dev/null | sort -rh | head -20
```

**Phase 2: Identify What's Consuming Space**

I'd look at the biggest consumers:

```bash
# Check Docker specifically
docker system df
docker system df -v

# Find large files
find / -type f -size +100M -exec ls -lh {} \; 2>/dev/null | sort -k5 -rh

# Check for deleted files still held open by processes
lsof +L1 2>/dev/null
```

This last command is critical — a very common scenario is that a log file was deleted (perhaps by a cleanup script) but a process still has it open, so the space isn't actually freed. The file is invisible in `du` but the space is consumed.

**Phase 3: Quick Recovery**

If I find large deleted files held open by processes:
```bash
# Find the PID
lsof +L1 | awk '{print $2}' | sort -u

# Truncate the file descriptor (frees space without killing the process)
> /proc/<PID>/fd/<FD_NUMBER>
```

If Docker logs are the culprit (extremely common):
```bash
# Truncate container logs
truncate -s 0 /var/lib/docker/containers/<container-id>/<container-id>-json.log

# Set a log rotation limit
```

If old Docker images/layers are consuming space:
```bash
# Remove stopped containers
docker container prune -f

# Remove unused images
docker image prune -a -f

# Nuclear option if desperate
docker system prune -a -f --volumes
```

**Phase 4: Root Cause Analysis**

After recovery, I'd investigate what caused the fill:
- Was it application logs growing unexpectedly?
- Was it a debug mode left enabled?
- Was it a temporary processing job that created large files?
- Was it Docker image layer accumulation from frequent builds?
- Was it a database growing beyond expected size?

**Phase 5: Permanent Fixes**

Implement logrotate for application and Docker logs, set up disk usage alerts at 80% and 90%, add Docker log rotation configuration, implement disk quotas, and schedule regular Docker system cleanup.

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                 prod-app-server-03                  │
│                                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │ API GW   │  │ Auth Svc │  │ Payment Service  │  │
│  │ Container│  │ Container│  │ Container        │  │
│  └────┬─────┘  └────┬─────┘  └────────┬─────────┘  │
│       │              │                  │            │
│  ┌────▼──────────────▼──────────────────▼─────────┐  │
│  │            Docker Overlay2 Storage             │  │
│  │  /var/lib/docker/                              │  │
│  │    ├── containers/<id>/<id>-json.log  ← GROWTH │  │
│  │    ├── overlay2/ (image layers)                │  │
│  │    └── volumes/                                │  │
│  └────────────────────────────────────────────────┘  │
│                                                     │
│  ┌────────────────────────────────────────────────┐  │
│  │          Disk: /dev/sda1  (50GB)              │  │
│  │          Usage: 100% ← CRITICAL               │  │
│  └────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

## Investigation

1. **Check overall disk status**: Run `df -h` to confirm which filesystem is full and how much space is available.
2. **Check inode usage**: Run `df -i` — disk can appear full due to inode exhaustion from millions of tiny files.
3. **Identify top-level consumers**: Run `du -sh /* 2>/dev/null | sort -rh | head -20` to find which directories are largest.
4. **Drill into Docker**: Run `docker system df` to see how much Docker is consuming in images, containers, build cache, and volumes.
5. **Check for deleted-but-open files**: Run `lsof +L1 2>/dev/null` — this catches the "phantom disk usage" scenario.
6. **Check application logs inside containers**: Inspect each container's log file size with `ls -lh /var/lib/docker/containers/<id>/<id>-json.log`.
7. **Check for large core dumps**: Run `find / -name "core.*" -o -name "*.core" 2>/dev/null` — core dumps can be gigabytes.
8. **Check database files**: If any container runs a database, check the data directory size.

## Commands

```bash
# 1. Overall disk usage
df -h

# 2. Inode usage (can be full even when blocks aren't)
df -i

# 3. Find the biggest directories
du -sh /* 2>/dev/null | sort -rh | head -20

# 4. Docker disk usage breakdown
docker system df -v

# 5. Find large files anywhere on the system
find / -type f -size +50M -exec ls -lh {} \; 2>/dev/null | sort -k5 -rh | head -30

# 6. Find deleted files still held open
lsof +L1 2>/dev/null | awk 'NR>1 {print $1, $2, $7, $9}' | sort -k3 -rh

# 7. Check Docker container log sizes
for cid in $(docker ps -q); do
  log=$(docker inspect --format='{{.LogPath}}' $cid)
  echo "$(du -sh $log 2>/dev/null) - $(docker inspect --format='{{.Name}}' $cid)"
done | sort -rh

# 8. Truncate a Docker container log without restarting
truncate -s 0 /var/lib/docker/containers/<container-id>/<container-id>-json.log

# 9. Safely truncate a file descriptor held by a running process
# First find the FD number
lsof +L1 | grep deleted
# Then truncate it
: > /proc/<PID>/fd/<FD_NUMBER>

# 10. Clean up Docker (safe to run)
docker container prune -f
docker image prune -a -f --filter "until=168h"  # Remove images older than 7 days
docker volume prune -f  # Be careful with this one
docker system prune -a -f --volumes  # Nuclear option

# 11. Find and remove old log files
find /var/log -name "*.gz" -mtime +30 -delete
find /var/log -name "*.old" -mtime +30 -delete

# 12. Check for large core dumps
find / -maxdepth 4 -name "core.*" -o -name "coredump" 2>/dev/null
```

## Root Cause

| Root Cause | Detection | Elimination |
|---|---|---|
| Docker container logs growing unbounded | `docker system df`, check log file sizes | Configure `max-size` and `max-file` in Docker daemon or docker-compose |
| Deleted files held open by processes | `lsof +L1` shows files with (deleted) suffix | Truncate via `/proc/PID/fd/FD` or restart the holding process |
| Debug logging enabled accidentally | Check application config for log level changes | Revert config, implement change control for log levels |
| Docker image layer accumulation | `docker system df` shows high image usage | Implement CI/CD cleanup, scheduled `docker image prune` |
| Large core dumps from crashes | `find / -name "core.*"` | Configure `core_pattern` to limit dump size, fix the crashes |
| Database data growth | Check DB data directory size | Implement data archival, partitioning, or scaling |
| Temporary files not cleaned up | `find /tmp -size +100M` | Implement cleanup cron jobs, set tmpwatch |

## Immediate Mitigation

1. **Truncate the largest log files** immediately to regain space (this doesn't lose historical data if the log is being written to the same file).
2. **Run `docker container prune -f`** to remove stopped containers.
3. **Run `docker image prune -a -f`** to remove unused images.
4. **Delete old system logs**: `journalctl --vacuum-size=500M` and remove rotated log files older than 7 days.
5. **If still critical**: Truncate the active application log via the `/proc/PID/fd/` method to reclaim space while the application continues running.
6. **Communicate status** to stakeholders with an ETA for full resolution.

## Permanent Fix

1. **Configure Docker log rotation** in `/etc/docker/daemon.json`:
```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "50m",
    "max-file": "3"
  }
}
```
2. **Implement logrotate** for all application logs in `/etc/logrotate.d/`.
3. **Set up automated Docker cleanup** cron job:
```bash
0 3 * * * /usr/bin/docker system prune -a -f --filter "until=168h" >> /var/log/docker-cleanup.log 2>&1
```
4. **Implement disk usage monitoring** with alerts at 80% (warning) and 90% (critical).
5. **Add inode monitoring** alongside disk block monitoring.
6. **Implement disk quotas** using `quota` or container resource limits.
7. **Use a centralized logging system** (ELK, Loki, CloudWatch) so logs don't accumulate on local disk.

## Monitoring

- **Disk usage**: Alert at 80% (warning) and 90% (critical) using Prometheus/node_exporter + AlertManager or similar.
- **Inode usage**: Alert when inode utilization exceeds 80%.
- **Docker disk usage**: Monitor `docker system df` metrics.
- **Log growth rate**: Track the rate of log file growth; alert if it exceeds normal baselines.
- **Application-level**: Monitor for "No space left on device" errors in application logs.
- **File descriptor usage**: Monitor `/proc/sys/fs/file-nr` for file handle exhaustion.
- **Scheduled cleanup verification**: Alert if Docker cleanup cron job hasn't run successfully.

## Security

- **Disk fill DoS**: A compromised service could intentionally fill the disk. Implement per-container disk quotas.
- **Log injection**: Attackers may inject verbose logging to trigger disk fills. Validate and sanitize log inputs.
- **Privilege escalation**: Truncating files via `/proc/PID/fd/` requires appropriate permissions. Ensure only authorized users can do this.
- **Docker socket exposure**: If `/var/lib/docker` fills up, check if any container has Docker socket mounted (security risk).
- **Sensitive data in logs**: When truncating logs, be aware they may contain sensitive data. Ensure centralized logging has proper access controls.

## Production Considerations

- **HA**: If this server is behind a load balancer, drain traffic before making changes. If it's a standalone server, plan for brief downtime.
- **Data integrity**: Ensure truncating logs doesn't lose critical audit trail data. Ship logs to centralized storage first if possible.
- **Cattle vs Pets**: If this is a "pet" server, invest in fixing the root cause. If it's "cattle," consider replacing it entirely.
- **Capacity planning**: Review if the disk was undersized for the workload. Consider moving to a larger volume or cloud-based storage.
- **Change management**: Document the incident, create a post-mortem, and feed findings back into infrastructure improvements.
- **Cost**: Compare cost of larger disks vs. centralized logging vs. engineering time for cleanup automation.
- **Compliance**: Ensure audit logs aren't lost during emergency cleanup. Maintain compliance with data retention policies.

## Senior-Level Answer

"I'd start by confirming the blast radius and communicating status, then immediately identify the largest space consumers using `du` and `docker system df`. The most common culprits in Docker environments are unbounded container logs and accumulated image layers. I'd truncate the largest log files using `/proc/PID/fd/` to avoid restarting containers, run `docker system prune` to clean up unused resources, and implement Docker log rotation at both the daemon level and via logrotate. For permanent prevention, I'd set up disk usage alerts, automated Docker cleanup, and investigate whether centralized logging is needed to move logs off the local disk entirely."

## Architect-Level Answer

"This incident reveals three systemic gaps: insufficient observability, missing guardrails, and architectural debt. First, we need proactive disk monitoring with graduated alerts — warning at 80%, critical at 90% — with automated escalation to prevent 100% utilization. Second, we need guardrails: Docker log rotation configured at the daemon level, per-container disk quotas, and automated cleanup jobs. Third, architecturally, we should evaluate whether local disk logging is appropriate for a production system at all — migrating to centralized logging (ELK, Loki, or cloud-native solutions) eliminates this class of problem entirely. I'd also recommend implementing immutable infrastructure patterns where servers are replaced rather than patched, which would make this a non-event — the old server gets drained and a fresh one with proper configuration replaces it. Finally, this should be a post-mortem with action items tracked to completion, including review of whether the server's disk capacity matches the workload's growth projections."

## Follow-Up Questions

1. "What's the difference between disk space exhaustion and inode exhaustion, and how would you handle each differently?"
2. "How would you handle this situation if the full disk was caused by a runaway Docker build cache during a CI/CD pipeline execution?"
3. "If the application uses a local SQLite database that's grown to 200GB, and you can't delete any data, what are your options?"
4. "How would you design a log rotation strategy for a microservices architecture where each service has different log volume and retention needs?"
5. "What would you do differently if the server was an EC2 instance with an EBS volume versus a bare-metal server with local storage?"
