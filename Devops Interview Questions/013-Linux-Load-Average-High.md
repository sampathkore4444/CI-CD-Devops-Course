# 13. High Load Average but Low CPU Usage

## Scenario
It's 3:40 PM on a Friday. Users report the application is "very slow" — page loads taking 10+ seconds. You check the monitoring dashboard and see that prod-worker-03 has a load average of 52.7 with only 8 CPU cores. However, CPU utilization is only 22%, and memory is at 40% usage. The iowait is low, and the network is normal. The server runs: a Redis cache, a Celery worker cluster (20 workers), nginx, and a Python Django application. The unusual thing is that the server is showing high load but the CPU isn't doing much work. You need to figure out what's causing the high load average and resolve it.

## Interviewer Question
"The server load average shows 50+ but CPU utilization is only 20%. Applications are extremely slow. You need to understand what's causing the high load and resolve it."

## What I Should Think About
- **What load average actually means**: The number of processes waiting in the run queue — includes processes in uninterruptible sleep (D state)
- **D-state processes**: Uninterruptible sleep from I/O, kernel operations, or blocking syscalls
- **Process states**: R (running), D (uninterruptible sleep), S (sleeping), T (stopped), Z (zombie)
- **Load vs CPU %**: Load average is NOT CPU utilization
- **Counting processes**: `ps -eo stat` can show how many are in R and D states
- **Common causes**: NFS hangs, fork bombs, high thread pool with waiting threads, kernel issues, mutex/contention

## Ideal Answer

**Phase 1: Understand What the Load Really Is**

First, let's distinguish between processes in the R state (actually running/waiting for CPU) and D state (uninterruptible sleep — usually waiting on I/O that can't be interrupted):

```bash
# Check load average
uptime
cat /proc/loadavg

# Count processes by state
ps -eo stat | awk '{if ($1 ~ /R/) r++; if ($1 ~ /D/) d++; if ($1 ~ /S/) s++} END {print "Running/Runnable:", r, "Interruptible-sleep:", s, "Uninterruptible-sleep:", d}'

# Or more detailed
ps -eo state,comm | awk '{count[$1]++} END {for (s in count) print s, count[s]}'

# Compare load components
cat /proc/loadavg
# Output: 52.77 48.30 42.15 12/1029 28977
# 52.77 = 1-min load, 12/1029 = 12 running/total, 28977 = last PID
```

**Phase 2: Identify the D-state Processes**

If there are hundreds of D-state processes, something is blocking them at the kernel level:

```bash
# Find D-state processes
ps -eo pid,stat,wchan,cmd | awk '$2 ~ /D/ {print}' | head -30

# The wchan column shows which kernel function they're waiting in
ps -eo pid,stat,wchan:40,cmd | head -50

# Check what syscalls they're blocked on
for pid in $(ps -eo pid,stat | awk '$2 ~ /D/ {print $1}' | head -10); do
  echo "PID: $pid"
  cat /proc/$pid/stack 2>/dev/null | head -5
  cat /proc/$pid/status | grep -E "State|Voluntary|nonvoluntary"
done
```

**Phase 3: Check for Common Culprits**

```bash
# Check NFS mounts (very common cause of D-state)
mount | grep nfs
df -h  # If NFS mounts hang, df hangs too

# Check for I/O issues
iostat -xz 1 5
mpstat 1 5

# Check for kernel lock contention
cat /proc/lock_stat 2>/dev/null | head -20

# Check for swap thrashing
vmstat 1 5

# Check the run queue over time
mpstat 1 5 | tail -5
```

**Phase 4: Investigate the Application**

If the D-state count isn't the cause, the load could be from a large number of runnable threads:

```bash
# Count threads in the run queue
ps -eLo stat,tid | awk '{if ($1 ~ /R/) r++} END {print "Runnable threads:", r}'

# Check Celery worker threads
ps -eLf | grep celery | wc -l
# If Celery has 20 workers with 4 threads each = 80 threads

# Check thread states per process
for pid in $(ps -eo pid,comm | grep -E "celery|python" | awk '{print $1}' | head -5); do
  echo "PID: $pid Threads in run queue:"
  ps -eLo pid,tid,stat | awk -v p=$pid '$1 == p && $3 ~ /R/ {print}' | wc -l
done

# Check for thread pool exhaustion
grep -r "thread" /etc/nginx/nginx.conf
```

**Phase 5: Resolve**

Based on findings:
- If NFS hangs: Remount or unmount the NFS share, move that workload off NFS
- If thread contention: Adjust thread pool sizes, worker counts
- If it's the run queue: Scale horizontally, reduce worker count
- If it's a kernel issue: Check for known bugs with the current kernel version

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│                  prod-worker-03                           │
│                  CPU: 8 cores                            │
│                  Load average: 52.7                      │
│                                                          │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────┐   │
│  │ Redis       │  │ Celery       │  │ Django App    │   │
│  │ 5% CPU      │  │ 15% CPU      │  │ 5% CPU        │   │
│  │ Load: low   │  │ Load: HIGH   │  │ Load: normal  │   │
│  └─────────────┘  └──────┬───────┘  └───────────────┘   │
│                          │                              │
│              ┌───────────▼───────────┐                  │
│              │ 20 Celery workers     │                  │
│              │ Each worker: 4 threads │                  │
│              │ = 80 threads total    │                  │
│              │                       │                  │
│              │ 40 threads in D state │  ← UNINTERRUPTIBLE│
│              │ waiting for...        │                     │
│              │ [wchan: pipe_wait,    │                     │
│              │  futex_wait,          │                     │
│              │  generic_borg_gather  │                     │
│              │  wait_for_common]     │                     │
│              └───────────────────────┘                  │
│                                                          │
│  Root cause found:                                       │
│  • 40 threads blocked on NFS (pipe_wait)                │
│  • Network Filesystem to NAS was hung                   │
│  • This creates "phantom load" — no CPU work,           │
│    but processes contribute to load average             │
└──────────────────────────────────────────────────────────┘
```

## Investigation

1. **Check the load average**: Run `uptime` to see the 1, 5, and 15-minute load averages.
2. **Compare load to CPU**: Run `mpstat 1 5` — if load is high but CPU is low, look for D states.
3. **Count processes by state**: Run `ps -eo stat | sort | uniq -c` to see how many processes are in R, D, S states.
4. **Find D-state processes**: Run `ps -eo pid,stat,wchan,cmd | awk '$2 ~ /D/'` — processes in D state indicate kernel-level blocking.
5. **Check the wchan column**: This shows which kernel function the process is waiting in. `pipe_wait`, `futex_wait`, and NFS functions are common culprits.
6. **Check NFS mounts**: Run `mount | grep nfs` and `df -h` — a hung NFS server causes massive D-state accumulation.
7. **Check for other system issues**: Run `iostat -xz 1 5` and `vmstat 1 5` to rule out disk and swap issues.
8. **Examine application process counts**: Check if a specific application is spawning excessive threads or workers.

## Commands

```bash
# 1. Check load average
uptime
cat /proc/loadavg

# 2. Check CPU stats
mpstat 1 5
top -bn1 | head -15

# 3. Count processes by state
ps -eo stat | sort | uniq -c
ps -eo state,comm | awk '{count[$1]++} END {for (s in count) print s, count[s]}'

# 4. Find D-state processes (uninterruptible sleep)
ps -eo pid,ppid,stat,wchan:50,cmd | awk '$3 ~ /D/' | head -40

# 5. Check what kernel functions D-state processes are waiting on
ps -eo stat,wchan | sort | uniq -c | sort -rn | head -20

# 6. Check process stacks for blocked processes
for pid in $(ps -eo pid,stat | awk '$2 ~ /D/' | head -5 | awk '{print $1}'); do
  echo "=== PID $pid ==="
  cat /proc/$pid/stack 2>/dev/null | head -10
  cat /proc/$pid/status 2>/dev/null | grep State
done

# 7. Check NFS mounts
mount | grep nfs
df -h 2>&1  # If this hangs, NFS is hung

# 8. Check disk I/O (to rule out disk-based D states)
iostat -xz 1 5

# 9. Check swap and memory
vmstat 1 5
free -h

# 10. Check thread counts
ps -eLf | wc -l
ps -eLf | awk '{count[$2]++} END {for (p in count) print p, count[p]}' | sort -k2 -rn | head -10

# 11. Check for file descriptor exhaustion
cat /proc/sys/fs/file-nr

# 12. Check for blocked pipes/FIFOs
find /proc/*/fd -type f -ls 2>/dev/null | grep -c pipe

# 13. Restart the hung NFS mount (if that's the cause)
umount -l /mnt/nfs_data   # Lazy unmount
mount -a                   # Remount everything
```

## Root Cause

| Root Cause | Detection | Resolution |
|---|---|---|
| Hung NFS mount | `df -h` hangs, many D-state processes on pipe_wait | Lazy unmount the NFS share, restore it after fixing the NFS server |
| I/O bottleneck (hidden) | `iostat -xz` shows high utilization | Fix the underlying storage issue |
| Thread pool explosion | Many threads in R state | Reduce thread pool size, fix thread exhaustion |
| Kernel bug/livelock | `dmesg` shows kernel messages, all cores busy in kernel | Update kernel, load kernel parameters |
| Swap thrashing | `vmstat` shows high si/so | Add memory, reduce memory consumption |
| Contention on locks (mutexes) | D-state on futex_wait | Fix the application's sync logic |
| Fork bomb / too many threads | Check `ps -eLf \| grep python` counts | Kill the cause, set ulimits |

## Immediate Mitigation

1. **If NFS is hung**: `umount -l /mnt/nfs` to lazily unmount the NFS share — this immediately releases the D-state processes.
2. **If no NFS issue**: Find the top blocked process with `ps -eo pid,stat,wchan`. Kill the root process if it's safe: `kill -9 <pid>`.
3. **Reduce worker/thread loads**: Reduce Celery workers: `systemctl stop celery-worker@5` (stop excess workers).
4. **Cap thread creation**: Set `ulimit -u` limits and adjust application configuration.
5. **Check if the load is actually improving**: Recheck load average after 5-10 minutes; it takes a while to drop.

## Permanent Fix

1. **Replace NFS with a reliable storage solution**: Consider GlusterFS, Ceph, or object storage with proper HA.
2. **Implement NFS mount timeout**: Add `timeo=30,retrans=5,soft` to NFS options so hangs aren't permanent.
3. **Try `loadavg` monitoring**: Alert on load average OR high D-state count, not just CPU usage.
4. **Configure thread limits**: Set `worker_prefetch_multiplier` and other Celery settings to prevent over-spawning.
5. **Implement D-state monitoring**: Alert when D-state process count exceeds a threshold.

## Monitoring

- **Load average**: Alert when load exceeds 1.5x the number of cores.
- **CPU vs load mismatch**: Alert when load is high but CPU is low — indicates D-state or scheduling issue.
- **D-state process count**: Alert when D-state processes exceed 10.
- **Per-process thread count**: Monitor for unexpected thread growth.
- **NFS mount health**: Monitor NFS mount responsiveness with periodic stat calls.
- **Application queue depth**: Monitor Celery queue length and worker utilization.

## Security

- **NFS security**: Ensure NFS mounts use proper authentication and firewall rules.
- **Load-based DoS**: A compromised process could create many heavy threads. Use ulimits and cgroups.
- **Kernel exploits**: High D-state counts can be caused by malicious kernel-level activity. Investigate unusual D-state wait channels.

## Production Considerations

- **HA**: If NFS is the cause, the NFS server itself needs HA to prevent this class of failure.
- **Scalability**: Worker processes should scale with queue depth, not stay at a fixed high count.
- **Cost**: Over-provisioning workers for a queue that doesn't need them wastes resources.
- **Application design**: The application should use async I/O and non-blocking operations to reduce D-state waits.
- **Operational**: Load average in isolation is misleading — always correlate with process states.

## Senior-Level Answer

"The key insight is that load average isn't CPU utilization. I'd check `ps -eo stat | sort | uniq -c` to count process states. If there are numerous D-state processes, they're in uninterruptible sleep — usually waiting on something like a hung NFS mount or kernel-level I/O. I'd check `ps -eo pid,stat,wchan,cmd | awk '$2 ~ /D/'` to see which kernel functions they're blocked on and check `mount | grep nfs` and `df -h` to verify NFS health. If NFS is hung, `umount -l` the mount immediately. If it's thread contention, I'd reduce worker thread counts. The fix depends entirely on what the wchan values reveal about what's blocking the processes."

## Architect-Level Answer

"High load with low CPU is a classic symptom of D-state process accumulation — usually from a storage dependency issue. The systemic solution involves several levels: First, eliminate the dependency on shared NFS in production by moving to a distributed storage solution with proper HA or cloud-native options. Second, implement intelligent worker scaling (Celery autoscale based on queue depth) instead of fixed worker counts that can amplify failures. Third, enhance our monitoring to detect 'phantom load' — we should be tracking per-process wait channels and D-state counts, not just the load average aggregate. Fourth, architect for resilience: services should use timeouts and circuit breakers so they don't hang indefinitely on a hung dependency. This incident should drive a dependency review: every external dependency (NFS, databases, APIs) should have defined failure modes and retry policies."

## Follow-Up Questions

1. "What exactly does the Linux load average represent? How does the kernel calculate it?"
2. "Explain uninterruptible sleep (D state). Why does the kernel use it, and what kernel functions cause it?"
3. "What is the difference between load average and CPU utilization? Can you construct a scenario where load is 50 on an 8-core machine but CPU is idle?"
4. "Why does a hung NFS mount produce D-state processes even though network operations are interruptible at the syscall level?"
5. "Design a comprehensive load monitoring system that differentiates between CPU-bound, I/O-bound, and network-bound load."