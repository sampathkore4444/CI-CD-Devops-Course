# 7. Disk I/O Bottleneck Causing Application Slowness

## Scenario
At 2:30 PM on a Wednesday, the application monitoring shows response times have degraded from an average of 50ms to 2.1 seconds. The server is a production node running a data processing pipeline that uses a local NVMe SSD for temporary data processing. CPU usage is normal at 35%, memory usage is at 60%, but `iowait` has spiked to 85%. The server runs several Java microservices and a Python-based ETL pipeline. The ETL pipeline was upgraded last night to process 10x more data. The `iostat` output shows the disk is doing 15,000 IOPS with an average wait time of 50ms. You need to identify what's causing the I/O bottleneck and resolve it without stopping the application.

## Interviewer Question
"Application response time has increased from 50ms to 2 seconds. CPU and memory are normal, but iowait is extremely high. The application uses a local SSD for temporary data processing. How do you investigate and resolve this disk I/O bottleneck?"

## What I Should Think About
- **I/O wait vs CPU**: `iowait` means processes are waiting for disk I/O
- **Which process is causing I/O**: Use `iotop` or `pidstat -d`
- **Type of I/O**: Sequential vs random, read vs write
- **Disk saturation**: Is the disk at its IOPS limit?
- **Page cache**: Is the system running out of page cache?
- **Swap thrashing**: Is the system swapping heavily?
- **Database queries**: Could be unindexed queries causing table scans
- **tmpfs vs disk**: Could temporary data use RAM-backed storage?

## Ideal Answer

**Phase 1: Confirm the I/O Bottleneck**

```bash
# Check iowait
mpstat 1 5

# Check disk utilization
iostat -xz 1 5

# Check overall system activity
vmstat 1 5
```

The `iostat` output shows:
```
Device   r/s    w/s   rMB/s  wMB/s  await  svctm  %util
nvme0n1  8500  6500  120.0  85.0   50.2   0.04   98.5
```

98.5% utilization confirms the disk is saturated. The high await time (50ms) confirms processes are waiting.

**Phase 2: Identify the Process Causing I/O**

```bash
# Top I/O consumers
iotop -oP -d 1

# Or using pidstat
pidstat -d 1 10

# Check page cache usage
free -h

# Check swap activity
vmstat 1 5  # Look at the 'si' and 'so' columns
```

The output shows:
```
PID    DISK READ  DISK WRITE  COMMAND
12345  125.0 MB/s  90.0 MB/s   python3 /opt/etl/process_data.py
```

The ETL process is consuming most of the I/O.

**Phase 3: Understand What the Process is Doing**

```bash
# What files is it reading/writing?
lsof -p 12345 | grep -E "\.csv|\.parquet|\.tmp|\.log"

# Check its strace briefly
timeout 5 strace -e trace=open,read,write -p 12345 2>&1 | head -50

# Check the ETL script's configuration
cat /opt/etl/config.yaml | grep -E "batch_size|buffer_size|concurrency"
```

**Phase 4: Mitigate the Bottleneck**

```bash
# Option 1: Reduce I/O priority of the ETL process
ionice -c2 -n7 -p 12345  # Best-effort, lowest priority

# Option 2: Limit the ETL's I/O using cgroups
sudo cgcreate -g blkio:/etl_limited
sudo cgset -r blkio.throttle.read_bps_device="259:0 52428800" /etl_limited  # 50MB/s
sudo cgset -r blkio.throttle.write_bps_device="259:0 52428800" /etl_limited
sudo cgclassify 12345 /etl_limited

# Option 3: Reduce batch size in the ETL configuration
sed -i 's/batch_size: 10000/batch_size: 1000/' /opt/etl/config.yaml

# Option 4: Throttle using ionice and renice
renice +19 -p 12345
ionice -c3 -p 12345  # Idle priority
```

**Phase 5: Long-term Solutions**

- Move temporary data processing to a separate dedicated disk
- Use tmpfs/RAM-backed storage for small temporary datasets
- Implement I/O scheduling improvements
- Optimize the ETL pipeline for I/O efficiency (batching, buffering)
- Consider async I/O or memory-mapped files

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│                  prod-worker-01                           │
│                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────┐  │
│  │ Java API     │  │ Java API     │  │ Python ETL     │  │
│  │ (normal I/O) │  │ (normal I/O) │  │ (I/O hog)     │  │
│  │ 2% disk     │  │ 3% disk     │  │ 95% disk  ←    │  │
│  └──────────────┘  └──────────────┘  └───────┬────────┘  │
│                                              │          │
│  ┌───────────────────────────────────────────▼────────┐  │
│  │ NVMe SSD (500GB)                                  │  │
│  │ Utilization: 98.5%                                │  │
│  │ IOPS: 15,000 (limit: 16,000)                     │  │
│  │ Read: 120 MB/s    Write: 85 MB/s                 │  │
│  │ Await: 50ms       Queue depth: 128               │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │ iowait: 85%  → Processes waiting for disk I/O     │  │
│  │ Page cache: 2GB used / 32GB total                  │  │
│  │ Swap: 0B used                                      │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

## Investigation

1. **Confirm I/O bottleneck**: Run `mpstat 1 5` to check iowait percentage and `iostat -xz 1 5` to see disk utilization, IOPS, and await times.
2. **Identify I/O-heavy processes**: Run `iotop -oP -d 1` or `pidstat -d 1 10` to find which process is consuming the most disk I/O.
3. **Check page cache and swap**: Run `free -h` and `vmstat 1 5` to see if the system is swapping (which adds I/O).
4. **Examine the offending process's files**: Run `lsof -p <pid>` to see which files it's reading and writing.
5. **Check disk queue depth**: Run `iostat -xz 1 5` and look at the `avgqu-sz` column — a high queue depth means the disk can't keep up.
6. **Check I/O scheduler**: Run `cat /sys/block/nvme0n1/queue/scheduler` to see which I/O scheduler is in use.
7. **Check for memory pressure causing page cache eviction**: Run `vmstat 1 5` — if `free` column is low, the page cache is being evicted and causing more disk reads.
8. **Profile the I/O pattern**: Use `blktrace` for deep analysis: `blktrace -d /dev/nvme0n1 -o - | blkparse -i -` to understand read/write patterns.

## Commands

```bash
# 1. Check I/O wait
mpstat 1 5

# 2. Check disk statistics
iostat -xz 1 5

# 3. Find top I/O processes
iotop -oP -d 1
pidstat -d 1 10

# 4. Check page cache
free -h
cat /proc/meminfo | grep -E "Cached|Buffers|Active\(file\)|Inactive\(file\)"

# 5. Check swap activity
vmstat 1 5  # Columns si (swap in), so (swap out)

# 6. Check what files a process has open
lsof -p <pid> | grep -E "\.csv|\.parquet|\.tmp|\.db"

# 7. Reduce I/O priority of a process
ionice -c2 -n7 -p <pid>  # Best-effort, lowest priority
ionice -c3 -p <pid>       # Idle priority

# 8. Check current I/O scheduler
cat /sys/block/nvme0n1/queue/scheduler

# 9. Change I/O scheduler (if needed)
echo mq-deadline > /sys/block/nvme0n1/queue/scheduler

# 10. Check disk queue depth and stats
iostat -xz 1 5 | grep nvme

# 11. Limit I/O using cgroups
sudo cgcreate -g blkio:/io_limited
sudo cgset -r blkio.throttle.read_bps_device="259:0 52428800" /io_limited
sudo cgclassify <pid> /io_limited

# 12. Use tmpfs for temporary data
mount -t tmpfs -o size=4G tmpfs /mnt/etl_temp
# Then point the ETL to use /mnt/etl_temp

# 13. Check for page cache thrashing
sar -B 1 5  # Page statistics

# 14. Use fio to benchmark disk
fio --name=randread --ioengine=libaio --iodepth=32 --rw=randread \
    --bs=4k --direct=1 --size=1G --numjobs=4 --runtime=60
```

## Root Cause

| Root Cause | Detection | Resolution |
|---|---|---|
| ETL process doing too much I/O | `iotop` shows high read/write rates | Limit I/O with ionice, cgroups, or batch size reduction |
| Unindexed database queries | `lsof` shows DB files being read heavily | Add indexes, optimize queries |
| Swap thrashing | `vmstat` shows si/so > 0 | Add RAM or reduce memory usage |
| Page cache too small | `free` shows minimal cache | Add RAM, tune vm.swappiness |
| I/O scheduler not optimal | Wrong scheduler for workload | Switch scheduler (mq-deadline, kyber, bfq) |
| Disk reaching end of life | `smartctl` shows errors | Replace the disk |

## Immediate Mitigation

1. **Reduce ETL I/O priority**: `ionice -c3 -p <ETL_PID>` — puts the process at idle I/O priority, giving other processes precedence.
2. **Limit ETL I/O bandwidth**: Use cgroups `blkio.throttle` to cap read/write rates.
3. **Reduce ETL batch size**: If the ETL has a configurable batch size, reduce it to lower I/O intensity.
4. **Move hot data to tmpfs**: If temporary data is small enough, mount a tmpfs and point the ETL there.
5. **Check for swap**: If swapping, `swapoff -a && swapon -a` to clear swap, then add more RAM.
6. **If all else fails**: Pause or kill the ETL process to restore application performance.

## Permanent Fix

1. **Separate I/O workloads**: Use a dedicated disk for the ETL pipeline vs. application data.
2. **Optimize the ETL pipeline**: Implement batching, buffering, and async I/O.
3. **Use appropriate storage**: NVMe for high-IOPS workloads, HDD for bulk storage.
4. **Implement I/O quotas**: Use cgroups to enforce I/O limits per process.
5. **Schedule heavy I/O during off-peak**: Move the ETL to run during low-traffic windows.
6. **Monitor I/O metrics**: Set up alerts for disk utilization and iowait.

## Monitoring

- **Disk utilization**: Alert when disk utilization exceeds 80% for 5+ minutes.
- **iowait**: Alert when iowait exceeds 30% for 5+ minutes.
- **Disk IOPS**: Monitor IOPS against the disk's rated capacity.
- **Await time**: Alert when average I/O wait time exceeds 20ms.
- **Queue depth**: Monitor disk queue depth for signs of saturation.
- **SMART data**: Monitor disk health for predictive failure.

## Security

- **I/O as attack vector**: A compromised process could intentionally saturate disk I/O as a DoS. Use cgroups to enforce I/O limits.
- **Data exposure**: Temporary files on disk may contain sensitive data. Use encrypted filesystems for sensitive temporary data.
- **Disk forensics**: If investigating an incident, preserve I/O logs before making changes.

## Production Considerations

- **HA**: Ensure other servers in the cluster can handle the load while this one is degraded.
- **SLA impact**: I/O bottlenecks directly impact user-facing response times. Escalate if SLA is at risk.
- **Capacity planning**: The 10x data increase should have triggered a capacity review before deployment.
- **Storage architecture**: Consider moving to a distributed storage system (Ceph, EBS) for better isolation.
- **Cost**: NVMe SSDs with higher IOPS capacity cost more. Balance between performance and cost.

## Senior-Level Answer

"I'd first confirm the I/O bottleneck with `mpstat` and `iostat -xz` to see disk utilization, IOPS, and await times. Then I'd use `iotop` to identify the specific process — likely the recently upgraded ETL pipeline. I'd immediately reduce its I/O priority with `ionice -c3` and apply cgroup-based I/O throttling. I'd also check for swap thrashing with `vmstat` and page cache pressure with `free`. The permanent fix involves separating ETL I/O from application I/O on different disks, implementing I/O scheduling, and reducing the ETL batch size. Long-term, I'd recommend adding a dedicated ETL worker node with its own storage to eliminate I/O contention entirely."

## Architect-Level Answer

"This I/O bottleneck is a symptom of poor workload isolation — a batch processing workload is competing for the same storage resources as latency-sensitive application traffic. Architecturally, we should separate these workloads onto different storage tiers: application data on fast, low-latency NVMe, and batch processing on a separate, cost-effective storage tier. I'd recommend implementing a storage class taxonomy where each workload is assigned an appropriate storage tier with guaranteed IOPS. Additionally, the ETL pipeline should be refactored to use streaming/chunked processing instead of bulk I/O, and temporary data should use memory-mapped files or tmpfs where possible. For the infrastructure, I'd evaluate whether the workload should move to a containerized environment with storage QoS (like Kubernetes with local PVs and resource quotas) or to a cloud-based solution with auto-scaling storage performance."

## Follow-Up Questions

1. "Explain the Linux I/O schedulers: mq-deadline, kyber, and bfq. When would you use each one?"
2. "What is the difference between synchronous and asynchronous I/O? How does `io_uring` improve I/O performance?"
3. "How would you use `blktrace` and `btrace` to diagnose I/O patterns at the block device level?"
4. "Design a storage architecture for a system that needs both high-IOPS transactional writes and high-throughput analytical reads."
5. "How would you implement I/O rate limiting in a multi-tenant environment where one tenant's I/O shouldn't affect another?"
