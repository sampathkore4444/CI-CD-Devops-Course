# 4. Identifying and Managing Rogue Processes

## Scenario
It's 11:30 AM on a Thursday. The monitoring dashboard shows prod-worker-02's CPU at 94%. The server runs a mix of Python data processing scripts, a Node.js API, and a Go-based metrics exporter, all running as systemd services (not containers). The CPU has been at this level for about 45 minutes. The application isn't fully down, but response times have increased from 100ms to 1.5 seconds. You notice a Python process (`process_large_dataset.py`) is consuming 92% CPU by itself. This script is part of a nightly ETL job that somehow got triggered during business hours. You need to identify it, understand why it's running, and handle it without affecting the Node.js API and Go exporter.

## Interviewer Question
"A single process is consuming 90%+ CPU on a production server running multiple services. You need to identify it, understand why it's consuming so much CPU, and resolve it without affecting the other services. Walk me through your approach."

## What I Should Think About
- **Process identification**: How to find the exact PID and what it's doing
- **Process hierarchy**: Is it a child of another process? Was it spawned by cron?
- **Resource limits**: cgroups, ulimits, nice/renice
- **Safe termination**: SIGTERM vs SIGKILL, signal handling
- **Impact of killing it**: Will data be corrupted? Will a dependency fail?
- **Why was it running?**: Cron misconfiguration? Manual trigger? Application bug?
- **Preventing recurrence**: How to ensure this doesn't happen again

## Ideal Answer

**Phase 1: Identify the Process**

```bash
# Top CPU consumers
top -bn1 | head -15

# Detailed process info
ps aux --sort=-%cpu | head -10

# Get the specific PID and its parent
ps -eo pid,ppid,pcpu,comm,args --sort=-pcpu | head -10

# Check what spawned it
pstree -p <pid>
```

The output shows:
```
PID   PPID  %CPU  COMMAND  ARGS
8842  1     92.3  python   /usr/bin/python3 /opt/scripts/process_large_dataset.py --full-refresh
```

PPID is 1, meaning it was either started directly or its parent already exited (like a cron job).

**Phase 2: Understand What It's Doing**

```bash
# Check the process's open files
ls -l /proc/<pid>/fd

# Check what files it's reading/writing
lsof -p <pid> | head -30

# Check its resource usage
cat /proc/<pid>/status | grep -E "VmRSS|VmSize|Threads|voluntary_ctxt_switches"

# Check system calls (briefly)
strace -c -p <pid> -e trace=all &  sleep 10; kill %1

# Check if it was started by cron
grep -r "process_large_dataset" /var/spool/cron/
grep -r "process_large_dataset" /etc/cron*
```

**Phase 3: Determine the Impact of Killing It**

```bash
# Check if any other processes depend on it
ps -eo pid,ppid,comm | grep <pid>

# Check if it's writing to a database
lsof -p <pid> | grep -E "\.db|\.sqlite|socket"

# Check if it has any lock files
find /opt/scripts/ -name "*.lock" -o -name "*.pid"

# Check the script itself to understand what it does
head -50 /opt/scripts/process_large_dataset.py
```

**Phase 4: Handle It**

Since this is a data processing script that was triggered inappropriately during business hours:

```bash
# Option A: Lower its priority first (least disruptive)
renice +19 -p <pid>

# Option B: Limit its CPU using cgroups
# Create a cgroup with CPU limit
sudo cgcreate -g cpu:/limited
sudo cgset -r cpu.cfs_quota_us=200000 /limited  # 20% of one core
sudo cgexec -g cpu:/limited <pid>

# Option C: Suspend it with SIGSTOP
kill -SIGSTOP <pid>

# Option D: Gracefully terminate with SIGTERM
kill -SIGTERM <pid>

# Wait, if it doesn't stop:
kill -SIGKILL <pid>
```

**Phase 5: Prevent Recurrence**

```bash
# Fix the cron configuration
crontab -l  # Check if it was accidentally scheduled
# Remove the erroneous cron entry

# Add CPU limits to the script's execution environment
# Use nice/n-ionice for the script
nice -n 19 ionice -c3 /opt/scripts/process_large_dataset.py

# Set up a systemd override to limit CPU
# /etc/systemd/system/etl-worker.service.d/override.conf
[Service]
CPUQuota=50%
```

## Architecture

```
┌────────────────────────────────────────────────────────┐
│                 prod-worker-02                          │
│                                                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ Node.js API  │  │ Go Exporter  │  │ Python ETL   │  │
│  │ PID: 1234    │  │ PID: 2345    │  │ PID: 8842    │  │
│  │ CPU: 3%      │  │ CPU: 1%      │  │ CPU: 92% ←   │  │
│  │ Status: OK   │  │ Status: OK   │  │ Status: ROGUE│  │
│  └──────────────┘  └──────────────┘  └──────┬───────┘  │
│                                             │          │
│  ┌──────────────────────────────────────────▼───────┐  │
│  │ Process 8842 details:                            │  │
│  │ CMD: python3 process_large_dataset.py            │  │
│  │ Started: 10:45 AM (by cron)                     │  │
│  │ CPU: 92%, Memory: 340MB                         │  │
│  │ Open files: /data/input/*.csv (reading)          │  │
│  │         : /data/output/results.parquet (writing) │  │
│  └──────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────┘
```

## Investigation

1. **Find the top CPU consumer**: Run `top -bn1 | head -15` to identify the PID and process name.
2. **Get parent process info**: Run `ps -eo pid,ppid,pcpu,comm,args --sort=-pcpu` to find if it has a parent or was orphaned.
3. **Check process tree**: Run `pstree -p <pid>` to see if it spawned child processes.
4. **Inspect open files**: Run `lsof -p <pid>` to see what files/network connections it's using.
5. **Check how it was started**: Look in crontab, systemd, or supervisor configs: `grep -r "process_large_dataset" /etc/cron* /etc/systemd/ /etc/supervisor/`.
6. **Read the script**: Examine the script to understand what it's supposed to do and if it has a bug.
7. **Check for locks**: Look for lock files or PID files that indicate the script's expected lifecycle.
8. **Monitor resource trends**: Use `pidstat -p <pid> 1 10` to see if CPU is stable or growing.

## Commands

```bash
# 1. Find top CPU consumers
top -bn1 -o %CPU | head -15
ps aux --sort=-%cpu | head -10

# 2. Get detailed process info
ps -eo pid,ppid,pcpu,pmem,stat,comm,args --sort=-pcpu | head -10

# 3. Check process tree
pstree -p <pid>
pstree -ap <pid>  # Include arguments

# 4. Monitor a specific process
pidstat -p <pid> 1 10

# 5. Check what files the process has open
lsof -p <pid> | head -30
ls -la /proc/<pid>/fd/ | head -20

# 6. Check system calls (10 seconds max)
timeout 10 strace -c -p <pid>

# 7. Lower priority (make it less CPU-aggressive)
renice +19 -p <pid>

# 8. Limit CPU with cgroups
sudo cgcreate -g cpu:/rogue
sudo cgset -r cpu.cfs_quota_us=200000 /rogue  # 20% of one core
sudo cgclassify <pid> /rogue

# 9. Suspend the process (SIGSTOP)
kill -SIGSTOP <pid>

# 10. Resume the process (SIGCONT)
kill -SIGCONT <pid>

# 11. Graceful termination
kill -SIGTERM <pid>

# 12. Force kill (last resort)
kill -SIGKILL <pid>

# 13. Kill all child processes too
pkill -P <pid>  # Kill children first
kill -9 <pid>    # Then kill parent

# 14. Check cron for the process
crontab -l 2>/dev/null
grep -r "process_large_dataset" /etc/cron* 2>/dev/null
grep -r "process_large_dataset" /var/spool/cron/* 2>/dev/null

# 15. Set CPU quota via systemd
systemctl set-property <service> CPUQuota=50%

# 16. Add nice value to prevent future occurrences
nice -n 19 /opt/scripts/process_large_dataset.py
ionice -c3 /opt/scripts/process_large_dataset.py
```

## Root Cause

| Root Cause | Detection | Resolution |
|---|---|---|
| Cron job triggered during business hours | Check crontab entries and timestamps | Move cron job to off-hours, add flock for exclusive execution |
| Script bug causing infinite loop | Read the script, check for while loops without breaks | Fix the bug, add timeouts, add resource limits |
| Missing resource limits | No CPU/memory limits set on the process | Implement cgroups, nice values, systemd CPUQuota |
| Manual execution by mistake | Check shell history: `history \| grep process_large` | Add guard rails to prevent accidental execution |
| Unbounded data processing | Script processes entire dataset without batching | Add chunking, pagination, and progress tracking |

## Immediate Mitigation

1. **Lower priority first**: `renice +19 -p <PID>` — this immediately reduces CPU contention with other services.
2. **If still too aggressive**: `kill -SIGSTOP <PID>` — freezes the process without killing it, allowing you to investigate safely.
3. **If investigation shows it's safe to kill**: `kill -SIGTERM <PID>` — send SIGTERM first, wait 10 seconds, then `kill -SIGKILL <PID>` if needed.
4. **Verify other services recover**: Monitor response times of the Node.js API after the rogue process is handled.
5. **Check for side effects**: Ensure the partially-written output files are cleaned up or the operation can be resumed.

## Permanent Fix

1. **Implement resource limits** for all batch processes:
   - Use systemd `CPUQuota` and `MemoryMax`
   - Use `nice` and `ionice` for I/O-sensitive workloads
   - Implement cgroups for fine-grained control
2. **Add execution guards**: Use `flock` to prevent overlapping executions:
```bash
flock -n /var/lock/etl.lock /opt/scripts/process_large_dataset.py
```
3. **Schedule properly**: Move batch jobs to off-peak hours with proper monitoring.
4. **Add timeouts**: Implement timeout mechanisms in scripts: `timeout 3600 /opt/scripts/process_large_dataset.py`.
5. **Implement circuit breakers**: Add logic to abort if processing takes too long.
6. **Monitor and alert**: Set up alerts for batch process runtime exceeding expected duration.

## Monitoring

- **Per-process CPU usage**: Monitor individual process CPU and alert when exceeding thresholds.
- **System load**: Alert when load average exceeds 1.5x CPU core count.
- **Batch job duration**: Alert when ETL jobs exceed expected runtime.
- **Process count**: Alert when total process count exceeds baseline.
- **Response time monitoring**: Alert when API response time degrades.
- **Resource usage trends**: Track CPU usage patterns to detect anomalies early.

## Security

- **Process isolation**: Run batch jobs under dedicated service accounts with minimal privileges.
- **Resource limits as security**: CPU/memory limits prevent resource exhaustion attacks.
- **Script permissions**: Ensure batch scripts are not world-writable and owned by the correct user.
- **Audit logging**: Log all process executions with timestamps and triggering users.
- **Privilege escalation**: Ensure batch jobs don't run as root unless absolutely necessary.

## Production Considerations

- **HA**: If this server handles traffic, ensure other services have adequate headroom.
- **Data integrity**: If the batch job was processing data, ensure partial results are handled gracefully.
- **Monitoring gaps**: If this went undetected for 45 minutes, monitoring needs improvement.
- **Runbook creation**: Document this scenario for the team to handle similar incidents.
- **Load testing**: Ensure the server can handle batch jobs and production traffic simultaneously.

## Senior-Level Answer

"I'd identify the rogue process using `top` and `ps aux --sort=-%cpu`, then check its parent process and how it was launched using `pstree` and crontab inspection. I'd examine open files with `lsof` to understand what it's doing. For immediate mitigation, I'd `renice` it to +19 to reduce priority, then `SIGSTOP` it to freeze it while I investigate. If it was a cron job that ran inappropriately, I'd fix the schedule. For the script itself, I'd add resource limits via systemd `CPUQuota`, implement execution locks with `flock`, and add timeout mechanisms. The key principle is: lower priority first, freeze if needed, then terminate gracefully."

## Architect-Level Answer

"This incident exposes a lack of workload isolation and resource governance. In a well-architected system, batch processing and interactive workloads should never compete for the same resources. I'd recommend three layers of defense: First, infrastructure-level isolation — run batch workloads on dedicated nodes or in separate resource groups with hard limits. Second, application-level controls — implement timeouts, circuit breakers, and resource budgets in batch jobs. Third, operational controls — use orchestration (Kubernetes CronJobs, systemd slices) that enforce resource limits and scheduling constraints. Additionally, I'd implement a workload classification system where all jobs are tagged as interactive, batch, or background, and resource policies are applied automatically. For this specific server, I'd move batch processing to a separate worker pool and implement proper monitoring with anomaly detection to catch runaway processes within minutes, not 45 minutes."

## Follow-Up Questions

1. "What's the difference between SIGTERM, SIGKILL, SIGSTOP, and SIGCONT? When would you use each?"
2. "Explain how Linux cgroups v1 and v2 work. How would you use them to limit a process's CPU and memory?"
3. "How would you handle this if the rogue process was a kernel thread rather than a user-space process?"
4. "What is CPU throttling via CFS bandwidth control, and how does it differ from nice values?"
5. "Design a system that automatically detects and handles rogue processes in production."
