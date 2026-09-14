# 47. Pod OOMKilled in Production

## Scenario

The `analytics-engine` pod in the `data-processing` namespace keeps getting OOMKilled. It's a JVM-based service running Apache Spark with `-Xmx1536m`. The pod has a 2GB memory limit. It worked perfectly in staging with 4GB limits. The OOM happens sporadically — sometimes after 2 hours, sometimes after 8 hours, typically during peak traffic when the data pipeline processes large batches. The development team insists the JVM settings are correct and it works in staging. You need to determine if it's a JVM issue, a native memory leak, or a container memory limit problem.

## Interviewer Question

"A production pod is being OOMKilled repeatedly. The pod has 2GB memory limit. The application is a JVM-based service with -Xmx set to 1536m. It worked fine in staging with 4GB limits. How do you diagnose whether it's a JVM issue, a memory leak, or a container limit issue?"

## What I Should Think About

- JVM memory ≠ container memory. JVM uses heap (Xmx) + metaspace + thread stacks + direct buffers + JIT code cache + native allocations
- Container cgroup limit includes ALL memory: heap + non-heap + page cache + kernel overhead
- The `-XX:MaxRAMPercentage` approach is better than fixed `-Xmx` for containers
- JVM doesn't know about container memory limits unless using `-XX:+UseContainerSupport` (default since Java 10)
- Native memory leaks (off-heap) can cause OOM even with correct heap settings
- Spark has its own memory management (execution memory + storage memory)
- Need to check actual memory usage vs limit over time, not just current snapshot
- Staging with 4GB works because there's headroom; production with 2GB doesn't have that headroom

## Ideal Answer

"The key insight is that JVM memory is much more than just the heap. With `-Xmx1536m` and a 2GB container limit, you have only 512MB for everything else: metaspace (default unlimited but typically 200-400MB for large apps), thread stacks (each thread ~1MB), direct byte buffers, JIT code cache, and page cache. During peak loads, Spark's execution and storage memory plus the JVM non-heap memory can easily exceed 2GB.

First, I'd check the actual memory usage pattern over time, not just the current state. Use Prometheus/Grafana or `kubectl top` to see the memory growth curve. If it's a sawtooth pattern (grows, GC drops, grows again), it's normal JVM behavior. If it's a steady upward trend, there's a memory leak.

Second, I'd enable JVM native memory tracking with `-XX:NativeMemoryTracking=summary` to see exactly where memory is going. Run `jcmd <pid> VM.native_memory summary` inside the container.

Third, the staging vs production difference is telling: staging with 4GB never hits the cgroup limit, so the OOM never triggers. Production with 2GB is right at the edge. The fix is either increasing the container limit or tuning the JVM to use less memory. For Spark specifically, I'd also check if `spark.executor.memory` and `spark.memory.fraction` are configured correctly."

## Architecture

```
Container Memory Layout (2GB limit):

┌─────────────────────────────────────────────────┐
│              Container (cgroup limit: 2GB)       │
│                                                 │
│  ┌──────────────────────────────────────────┐   │
│  │          JVM Heap (-Xmx1536m)            │   │
│  │  ┌────────────┐  ┌────────────────────┐  │   │
│  │  │  Young Gen  │  │     Old Gen        │  │   │
│  │  │  (Eden+     │  │     (Long-lived    │  │   │
│  │  │   S0+S1)    │  │      objects)      │  │   │
│  │  └────────────┘  └────────────────────┘  │   │
│  └──────────────────────────────────────────┘   │
│                                                 │
│  ┌─────────────────────┐  ┌──────────────────┐  │
│  │  Metaspace           │  │  Thread Stacks    │  │
│  │  (~300MB typical)    │  │  (~100 threads    │  │
│  │  Class metadata      │  │   × 1MB = 100MB) │  │
│  └─────────────────────┘  └──────────────────┘  │
│                                                 │
│  ┌─────────────────────┐  ┌──────────────────┐  │
│  │  Direct Buffers      │  │  JIT Code Cache   │  │
│  │  (NIO/Spark)         │  │  (~240MB)         │  │
│  │  (~200MB during      │  │                   │  │
│  │   peak processing)   │  │                   │  │
│  └─────────────────────┘  └──────────────────┘  │
│                                                 │
│  ┌──────────────────────────────────────────┐   │
│  │  Spark Execution & Storage Memory         │   │
│  │  (Off-heap, managed by Spark)             │   │
│  │  spark.memory.fraction = 0.6              │   │
│  │  = 0.6 × 1536m = ~920MB                  │   │
│  └──────────────────────────────────────────┘   │
│                                                 │
│  ┌──────────────────────────────────────────┐   │
│  │  OS/Page Cache + Kernel Overhead (~200MB) │   │
│  └──────────────────────────────────────────┘   │
│                                                 │
│  Total Estimated: ~3460MB  >>>  2GB LIMIT!      │
└─────────────────────────────────────────────────┘

Sawtooth Memory Pattern (Normal):
Memory
  ▲
2GB│         ╱╲      ╱╲      ╱╲
   │        ╱  ╲    ╱  ╲    ╱  ╲     ← GC drops memory
   │       ╱    ╲  ╱    ╲  ╱    ╲
   │      ╱      ╲╱      ╲╱      ╲
1GB│─────╱────────────────────────╲─────
   │
   └──────────────────────────────────▶ Time

Memory Leak Pattern:
Memory
  ▲
2GB│ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─  ← Cgroup limit
   │              ╱╲     ╱╲    OOM!
   │             ╱  ╲   ╱  ╲  ↑
   │            ╱    ╲ ╱    ╲ │
   │           ╱      ╲      ╲│
1GB│──────────╱─────────────────────
   │
   └──────────────────────────────────▶ Time
     Each GC cycle doesn't return to baseline → LEAK
```

## Investigation

**Step 1: Check the OOM event and last state**

```bash
kubectl get pod analytics-engine-0 -n data-processing -o jsonpath='{range .status.containerStatuses[*]}{.name}: lastState={.lastState}, restartCount={.restartCount}{"\n"}{end}'
```

```bash
kubectl describe pod analytics-engine-0 -n data-processing | grep -A 5 "Last State"
```

**Step 2: Check memory usage trend (requires metrics-server)**

```bash
kubectl top pod analytics-engine-0 -n data-processing --containers
```

**Step 3: Enable JVM native memory tracking**

```bash
# Add to container args
-XX:NativeMemoryTracking=summary

# Then exec in and check
kubectl exec -it analytics-engine-0 -n data-processing -- jcmd 1 VM.native_memory summary
```

**Step 4: Check Spark-specific memory configuration**

```bash
kubectl exec -it analytics-engine-0 -n data-processing -- env | grep -i spark
kubectl exec -it analytics-engine-0 -n data-processing -- cat /spark/conf/spark-defaults.conf
```

**Step 5: Check dmesg for OOM killer details**

```bash
kubectl debug node/<node-name> -it --image=busybox -- dmesg | grep -i "oom\|killed"
```

**Step 6: Analyze the memory growth curve**

```bash
# Check Prometheus for memory usage over time
# PromQL: container_memory_working_set_bytes{pod="analytics-engine-0",namespace="data-processing"}
```

## Commands

```bash
# Check current memory usage
kubectl top pod analytics-engine-0 -n data-processing --containers

# Get pod resource limits
kubectl get pod analytics-engine-0 -n data-processing -o jsonpath='{.spec.containers[*].resources}'

# Check OOM count over time
kubectl get pod analytics-engine-0 -n data-processing -o jsonpath='{.status.containerStatuses[*].restartCount}'

# Enable native memory tracking (requires JVM flag)
kubectl exec -it analytics-engine-0 -n data-processing -- jcmd 1 VM.native_memory summary

# Check heap usage
kubectl exec -it analytics-engine-0 -n data-processing -- jcmd 1 GC.heap_info

# Check Spark executor memory
kubectl exec -it analytics-engine-0 -n data-processing -- cat /proc/1/cmdline | tr '\0' '\n'

# Check all JVM flags
kubectl exec -it analytics-engine-0 -n data-processing -- jcmd 1 VM.flags

# Monitor memory in real-time
kubectl exec -it analytics-engine-0 -n data-processing -- bash -c "while true; do free -m; sleep 5; done"

# Check cgroup memory limit
kubectl exec -it analytics-engine-0 -n data-processing -- cat /sys/fs/cgroup/memory/memory.limit_in_bytes

# Check actual memory usage from cgroup
kubectl exec -it analytics-engine-0 -n data-processing -- cat /sys/fs/cgroup/memory/memory.usage_in_bytes

# Check node dmesg for OOM events
kubectl debug node/<node-name> -it --image=busybox -- dmesg | grep -i "oom\|killed process"

# Get historical OOM events
kubectl get events -n data-processing --field-selector reason=OOMKilling --sort-by='.lastTimestamp'

# Check if memory limits were recently changed
kubectl rollout history deployment/analytics-engine -n data-processing
```

## Root Cause

| Root Cause | Evidence | How to Eliminate |
|---|---|---|
| **JVM Heap + Non-Heap Exceeds Limit** | OOM happens during peak, native memory tracking shows high non-heap | Increase container memory limit or reduce JVM flags |
| **Spark Off-Heap Memory** | OOM during batch processing, `spark.executor.memory` too high | Tune `spark.memory.fraction` and `spark.memory.storageFraction` |
| **Direct Buffer Leak** | Native memory shows growing "Internal" section, NIO usage high | Fix code that allocates DirectByteBuffers without releasing |
| **Metaspace Leak** | Metaspace grows continuously, class loading increases | Set `-XX:MaxMetaspaceSize`, fix classloader leak |
| **Thread Leak** | Thread count grows over time | Fix code creating threads without pool |
| **Container Limit Too Low** | All memory regions add up to >2GB | Increase memory limit to match actual usage |
| **JVM Not Using Container Support** | JVM reports more memory than container limit | Use Java 10+, ensure `-XX:+UseContainerSupport` |

## Immediate Mitigation

```bash
# 1. Increase memory limit temporarily
kubectl patch deployment analytics-engine -n data-processing --type='json' -p='[
  {"op": "replace", "path": "/spec/template/spec/containers/0/resources/limits/memory", "value": "4Gi"}
]'

# 2. Reduce JVM heap to fit within current limit
kubectl patch deployment analytics-engine -n data-processing --type='json' -p='[
  {"op": "replace", "path": "/spec/template/spec/containers/0/env/0/value", "value": "-Xmx1024m -XX:MaxMetaspaceSize=256m -XX:MaxDirectMemorySize=256m"}
]'

# 3. Restart the pod to clear any accumulated memory
kubectl delete pod analytics-engine-0 -n data-processing

# 4. Scale down processing to reduce memory pressure
kubectl patch statefulset analytics-engine -n data-processing --type='json' -p='[
  {"op": "replace", "path": "/spec/replicas", "value": 2}
]'
```

## Permanent Fix

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: analytics-engine
  namespace: data-processing
spec:
  replicas: 3
  selector:
    matchLabels:
      app: analytics-engine
  template:
    spec:
      containers:
      - name: analytics-engine
        image: registry.internal/analytics-engine:v3.8.2
        resources:
          requests:
            memory: "3Gi"
            cpu: "1000m"
          limits:
            memory: "4Gi"
            cpu: "2000m"
        env:
        - name: JAVA_OPTS
          value: >-
            -XX:+UseContainerSupport
            -XX:MaxRAMPercentage=75.0
            -XX:MaxMetaspaceSize=512m
            -XX:MaxDirectMemorySize=512m
            -XX:NativeMemoryTracking=summary
            -XX:+UseG1GC
            -XX:MaxGCPauseMillis=200
        - name: SPARK_EXECUTOR_MEMORY
          value: "2g"
        - name: SPARK_MEMORY_FRACTION
          value: "0.6"
        - name: SPARK_MEMORY_STORAGE_FRACTION
          value: "0.5"
```

## Monitoring

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: oom-killed-alerts
spec:
  groups:
  - name: oom.rules
    rules:
    - alert: PodOOMKilled
      expr: kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} > 0
      for: 0m
      labels:
        severity: critical
      annotations:
        summary: "Pod {{ $labels.pod }} was OOMKilled"

    - alert: PodMemoryNearLimit
      expr: container_memory_working_set_bytes / container_spec_memory_limit_bytes > 0.9
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.pod }} memory usage > 90% of limit"

    - alert: HighJVMHeapUsage
      expr: jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"} > 0.85
      for: 10m
      labels:
        severity: warning
```

## Security

- **Resource DoS**: Without memory limits, a single pod can consume all node memory, affecting other tenants
- **OOM as attack vector**: Malicious pods could intentionally trigger OOM to disrupt co-located services
- **Native memory tracking**: `-XX:NativeMemoryTracking=summary` adds ~5-10% overhead but is essential for debugging

## Production Considerations

- **VPA (Vertical Pod Autoscaler)**: Can automatically adjust memory limits based on actual usage patterns
- **Node autoscaler**: Ensure nodes have enough memory for the largest pod
- **Pod disruption budgets**: OOM kills don't count against PDB, so ensure enough replicas
- **Memory guarantees**: Set `requests` to the typical usage, `limits` to the maximum expected

## Senior-Level Answer

"I'd diagnose this by understanding that JVM memory is heap + non-heap (metaspace, thread stacks, direct buffers, JIT cache) + Spark off-heap memory. With `-Xmx1536m` and a 2GB container limit, there's only ~500MB for everything else, which isn't enough during peak processing. The staging environment works because it has 4GB — plenty of headroom. I'd enable JVM native memory tracking (`-XX:NativeMemoryTracking=summary`) and run `jcmd 1 VM.native_memory summary` to see exactly where the memory goes. The fix is either increasing the container limit to 3-4GB or using `-XX:MaxRAMPercentage=75.0` with reduced Spark memory fractions. I'd also check for direct buffer leaks in the Spark configuration, as these are off-heap and invisible to standard JVM monitoring."

## Architect-Level Answer

"At the architecture level, this is a classic staging-production parity problem combined with JVM container awareness. The fundamental issue is that the staging environment's 4GB limit masked a design problem. I'd implement: (1) Right-sizing via VPA recommendations — let Kubernetes tell you the actual memory needs, (2) Use `-XX:MaxRAMPercentage` instead of fixed `-Xmx` for container-native JVM memory management, (3) Build staging with production-equivalent resource limits using resource quotas, (4) Implement memory profiling in the CI pipeline with production-like data volumes, and (5) Consider using JVM container-aware flags (`-XX:+UseContainerSupport`, `-XX:ActiveProcessorCount`) as defaults in the base container image. The architectural decision should be: either give the workload the memory it actually needs (4GB) or redesign the Spark jobs to be more memory-efficient."

## Follow-Up Questions

1. "How does `-XX:MaxRAMPercentage` differ from `-Xmx` in a container environment?"
2. "Explain the difference between RSS (Resident Set Size) and working set memory in the context of Kubernetes OOM kills."
3. "What happens when JVM's garbage collector runs but the reclaimed memory isn't returned to the OS?"
4. "How would you set up automatic memory limit tuning using Kubernetes VPA?"
5. "What is the difference between cgroup v1 and cgroup v2 memory accounting, and how does it affect OOM kills?"
