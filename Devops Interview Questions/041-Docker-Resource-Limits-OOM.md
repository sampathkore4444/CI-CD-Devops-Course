# 041. Container Resource Limits Causing OOM Kills

## Scenario

A Java 11 Spring Boot microservice runs inside a Docker container. The deployment sets `--memory=2g` (2 GB limit) and `--cpus=2`. The JVM is allowed 4 GB heap via the app's startup script:

```
JAVA_OPTS="-Xms4g -Xmx4g"
```

The container gets OOM-killed within minutes of starting:

```
docker inspect <id> --format '{{.State.OOMKilled}}'
=> true
docker logs <id>
=> "There is insufficient memory for the Java Runtime Environment to continue."
=> "Native memory allocation (mmap) failed to map 12320768 bytes"
```

The app ran fine before resource limits were introduced "to control cost." Without the limit it uses up to 3.2 GB RSS and runs for weeks. The team doesn't understand why a "2 GB limit" kills an app that uses 3.2 GB — and why setting a higher heap (4 GB) inside a smaller limit makes no sense at all.

The on-call engineer says "the app always worked without limits, so limits are the problem — let's remove them."

## Interviewer Question

"A Java application container keeps getting OOM-killed. You set `--memory=2g` on the container, but the JVM heap is configured for `-Xmx4g`. The app runs without limits but crashes when you add them. Walk me through exactly what's happening inside the container — how cgroup memory accounting, the JVM heap, and native memory (metaspace, thread stacks, direct buffers) interact — and then show me the correct way to configure resource limits for a JVM in Docker, both the Docker flags and the JVM flags, with a formula for sizing."

## What I Should Think About

- **cgroup memory accounting is NOT just the heap.** Container RSS = JVM heap + metaspace + thread stacks + code cache + GC overhead + direct buffers + JIT + native libraries. `-Xmx4g` inside a `-m2g` container is physically contradictory.
- **Two failure mechanisms:**
  1. **Host/cgroup OOM killer** — cgroup usage hits the 2 GB cap → kernel SIGKILLs the container's process group → exit 137, `State.OOMKilled=true`.
  2. **JVM-internal native allocation failure** — JVM tries `mmap` to reserve heap at startup (`-Xms4g`) → the allocation exceeds cgroup `memory.max` → `mmap` fails → `Error occurred during initialization of VM` → JVM exits with its own error (exit code typically `1`), before the kernel OOM killer fires. This is why you see two *different* error signatures.
- **JVM container-awareness (the elephant):**
  - Since **JDK 8u191 / 8u192+ and JDK 10+**, `-XX:+UseContainerSupport` is *on* by default. The JVM reads cgroup limits from `/sys/fs/cgroup` (and the newer cgroup v2 `memory.max`) and sizes its heap accordingly — **only if you don't override with explicit `-Xmx`**.
  - If you pass an explicit `-Xmx4g`, container awareness is irrelevant — the JVM will *try* to use 4 GB heap, guaranteed.
  - Default (no `-Xmx`): the JVM picks `MaxRAMPercentage=25%` of the *container* limit (with UseContainerSupport), not the host. So `-m2g` → default max heap ≈ 500 MB — app survives but probably runs out of heap if it truly needed 3.2 GB.
- **The mismatch scenario in the question:** `-Xmx4g` under `-m2g` is a guaranteed crash *before the app even starts* (or on first real allocation). The only two correct resolutions are consistent with each other — align limits.
- **The sizing formula:**
  - Container limit ≥ heap + overhead overhead ≈ 1.5× heap or `heap ≤ 0.6–0.7 × limit`. I.e., `-m2g` → `-Xmx ~1200–1400m`; or `-Xmx4g` → `-m6g`+.
  - Use `-XX:MaxRAMPercentage=x` with x ≤ 60–70 so the heap never consumes the whole cgroup; CgroupV2 flags: `-XX:MaxRAMPercentage=60.0`, `-XX:MinRAMPercentage=50.0`, `-XX:InitialRAMPercentage=50.0`.
- **More accurate knobs:**
  - `-XX:MaxRAMPercentage`, `-XX:InitialRAMPercentage`, `-XX:MinRAMPercentage` (JDK 8u191+).
  - `-XX:+ExitOnOutOfMemoryError` to fail fast rather than hang.
  - `-XX:MaxMetaspaceSize`, cap JIT code cache `-XX:ReservedCodeCacheSize`, thread stack size `-Xss`, and `-XX:MaxDirectMemorySize` for NIO buffers — these are *native* memory you often forget in the budget.
  - In newer JDKs (17/21), JVM cgroup awareness is mature but **you still must align flags with the cgroup**.
- **What does "no limit" mean?** Without `-m`, container RSS can approach host RAM / other containers' contention → a real "host-health" incident. Removing limits isn't the fix when you need cost isolation; sizing properly is.
- **Correct investigation:** `docker inspect` → `HostConfig.Memory`, `docker stats` → actual RSS, `dmesg`/cgroup logs → which line OOM'd, and inside: `jcmd <pid> VM.native_memory` / `NMT` summary to see the real decomposition — heap vs non-heap.

## Ideal Answer

"This is a classic cgroup contract violation, and it has two distinct crash signatures depending on how the JVM dies.

**Why it crashes, precisely.** Docker's `--memory=2g` creates a cgroup (or writes `memory.max`, cgroup v2) capping the container's total RSS including file cache pages attributable to the cgroup. The JVM's `-Xmx4g` says "I want up to 4 GB just for the Java heap." The heap is only part of the picture: metaspace for classes, code cache for JIT, per-thread stacks, GC structures, NIO direct byte buffers, and the JIT compiler all consume *native* memory on top of the 4 GB heap. When the JVM at startup tries to commit 4 GB (`-Xms4g`) or the app allocates up toward 4 GB, the cgroup rejects the memory and the kernel OOM killer (or the failing `mmap`, if the reservation itself exceeds the cgroup) kills the container.

The two kill paths you'll observe:
1. JVM prints `Native memory allocation (mmap) failed...` — the JVM couldn't even *reserve* the heap → that's a startup failure (JVM exits on its own before running your `main`). This happens when the reservation (Xms) exceeds the cgroup limit.
2. `ExitCode=137`, `OOMKilled=true`, kernel `Out of memory: Killed process ... (java)` in `dmesg` — used memory exceeded the cap during runtime → SIGKILL.

**The correct configuration.** The container memory limit must be a *superset* of the JVM's total memory footprint, and the JVM must be told to stay inside the remaining envelope. The effective standard:

- **Rule: container limit ≥ heap + native overhead; heap ≈ 50–60% of the limit.**
  With `--memory=2g`:
  ```
  --memory=2g --cpus=2
  JAVA_OPTS="-XX:+UseContainerSupport -XX:MaxRAMPercentage=60.0 -XX:InitialRAMPercentage=50.0 -XX:+ExitOnOutOfMemoryError"
  ```
  → max heap ≈ 1.2 GB, native (~metaspace, code, threads) ≈ 600–700 MB available, ~100–200 MB headroom.
- Or keep the **4 GB heap** and set **`--memory=7g`** (heap 4 GB + ~2.5 GB native overhead + slack) — but code is more portable and fits cost budgeting better with the first pattern.
- **Never hardcode `-Xmx` in the Dockerfile when limits vary by environment.** Prefer `MaxRAMPercentage` and inject only `JAVA_OPTS` from the deployment, so staging (2g) and prod (8g) don't force image rebuilds or submit to the same DEFAULTS that killed this container.

**The investigation.** I'd confirm with `docker inspect` (`HostConfig.Memory == 2g`), `docker stats` (RSS before the crash), and `dmesg | grep -i oom` (which process/thread was killed). Inside, I'd get the *real* memory breakdown from the JVM itself: `jcmd <pid> VM.native_memory summary` (with `-XX:NativeMemoryTracking=summary` enabled) shows heap, metaspace, code cache, GC, thread stacks, and direct buffers. That tells me the true envelope instead of guessing "3.2 GB used" — I can see *where* the 3.2 GB came from.

**Why 'remove the limits' is the wrong fix.** Removing the cap to fix the JVM is admitting you can't budget the cost, and it lets one container balloon to a host-wide OOM affecting *every* tenant. The mature answer is aligning the contract: pick the limit and size the JVM inside it — with `MaxRAMPercentage`, native-memory caps, and headroom — and monitor the footprint so next week's leak surfaces in dashboards before it surfaces as a page."

## Architecture

```
JVM inside a cgroup: total memory "bill we must budget"

┌────────────────────────────────────────────────────────┐
│ cgroup memory.max = 2 GB  (--memory=2g)                │
│                                                        │
│  ┌──────────┬───────────┬──────────┬───────────────┐   │
│  │  Java    │ Metaspace │ JIT code │ thread stacks │   │
│  │  heap    │ (classes) │ cache    │ per thread    │   │
│  │ -Xmx4g ──┼─  ~100-150m ─┼ ~50-80m ┼─ n_threads×1m │   │
│  └──────────┴───────────┴──────────┴───────────────┘   │
│  + GC structures + direct/NIO buffers (MaxDirectMemory)│
│  + native libs                                          │
│                                                        │
│  TOTAL CLAIM ≈ 4 GB heap + 1-1.5 GB native             │
│  ── exceeds 2 GB cap on day one → SIGKILL/137          │
└────────────────────────────────────────────────────────┘

CORRECT (Option A: shrink JVM to fit 2 GB cap):
┌──────────────────────────────────────────────┐
│ --memory=2g --cpus=2                          │
│ JAVA_OPTS="-XX:MaxRAMPercentage=60.0 ..."    │
│ heap max ≈ 1.2 GB   =========================│──────┐
│ native (meta/code/threads/direct) ≈ 0.5 GB   │  budget
│ headroom ≈ 0.3 GB                            │──────┘
└──────────────────────────────────────────────┘

CORRECT (Option B: grow cap to fit 4 GB heap):
┌──────────────────────────────────────────────┐
│ --memory=7g --cpus=2                          │
│ -Xmx4g (explicit)          ─ 4 GB heap        │
│ + native + direct ≈ 2.5 GB                    │
│ + headroom                                    │
│ TOTAL ≈ 7 GB ≥ 4 GB heap contract             │
└──────────────────────────────────────────────┘

Key equation to teach in interviews:
   ContainerLimit ≥ HeapMax + NativeOverhead
   NativeOverhead ≳ 0.3 × HeapMax (often much more w/ direct buffers)
   HeapMax  ≤ 60% × ContainerLimit             (or -XX:MaxRAMPercentage=60)
```

## Investigation

1. **Confirm the kill signature.** `docker inspect <id> --format '{{.State.OOMKilled}} {{.State.ExitCode}}'`. `OOMKilled=true` + exit 137 = SIGKILL by (c)group memory cap; exit 1 + a JVM-printed `mmap failed` in logs = reservation exceeded at startup. Different fixes.
2. **Read the cgroup/host evidence.** On the Docker host:
   ```bash
   sudo dmesg | grep -i -E "killed process|oom"
   ```
   Look for a line `Killed process X (java) total-vm:...` — total-vm includes the *reserved* heap-then-native space; compare against the cgroup.
   Actually, the canonical signal on cgroup v2:
   ```bash
   cat /sys/fs/cgroup/.../memory.events   # oom_kill counter
   ```
3. **Verify the actual limit applied:**
   ```bash
   docker inspect <id> --format 'Limit={{.HostConfig.Memory}} Swap={{.HostConfig.MemorySwap}} CPUs={{.HostConfig.NanoCpus}}'
   ```
   A `MemorySwap=-1` means unlimited swap — that makes app "use more" without automatically dying when swap is available; if swap is disabled in Kernel, behavior differs.
4. **Measure the real footprint before it dies.** Bring the container up with *full* limits + the SAME JVM flags short of killing it — start with a generous cap (`--memory=6g`) and watch `docker stats <id>`: record RSS at steady state and during a load spike. That gives the *actual* native overhead rather than guessing.
5. **Get the JVM's own decomposition (NMT).** If the app is alive enough:
   ```bash
   docker exec <id> jcmd <pid> VM.native_memory summary
   ```
   Requires the container to run with `-XX:NativeMemoryTracking=summary`… on the next/fixed start. The summary lists `Java Heap`, `Metaspace`, `Code`, `Thread`, `GC`, `Compiler`.
6. **Check the JVM version/cgroup support:**
   ```bash
   docker exec <id> java -XX:+PrintFlagsFinal -version 2>&1 | grep -iE "MaxRAM|ContainerSupport|UseCGroup"
   ```
   Confirm `-XX:+UseContainerSupport` and what `MaxHeapSize` the JVM derived by default (if no `-Xmx`).
7. **Reconstruct the budget grid on paper.** List: heap (from `-Xmx`), metaspace cap, code cache, thread count × `-Xss` (Java default 512k–1m), direct NIO max, GC. Sum vs `-m`. This "line-item budget" is the artifact you keep for future sizing reviews.
8. **Repro in isolation to distinguish heap vs native:**
   - Lower heap only (`-Xmx1g`): if it now survives → heap was the offender.
   - Keep heap, lower threads/metaspace (`-Xss256k`, `-XX:MaxMetaspaceSize=128m`): if it still dies → native/direct.

## Commands

```bash
# 1. What did the box die of?
docker inspect <container_id> \
  --format 'OOMKilled={{.State.OOMKilled}} ExitCode={{.State.ExitCode}}'

# 2. Host kernel view of the death
# (cgroup v2) check memory events file for the container slice:
find /sys/fs/cgroup -path '*<container_id>*memory.events' -exec cat {} \;
# (cgroup v1 fallback) host-side:
sudo dmesg | tail -30 | grep -i -E "oom|killed process"

# 3. What limits were active
docker inspect <container_id> \
  --format 'mem={{.HostConfig.Memory}} swap={{.HostConfig.MemorySwap}} cpus={{.HostConfig.NanoCpus}}'

# 4. Live RSS trend (before kaput)
docker stats <container_id> --no-stream

# 5. JVM view of the limits (are we container-aware?)
docker exec <container_id> sh -c 'java -XX:+PrintFlagsFinal -version' 2>&1 \
  | grep -iE "UseContainerSupport|MaxHeapSize|MaxRAMPercentage|ActiveProcessorCount"

# 6. Native memory breakdown (requires NMT enabled at JVM start)
docker exec <container_id> jcmd <pid> VM.native_memory summary

# 7. Corrected start for a 2 GB container (heap capped by percentage)
docker run -d --name app \
  -m 2g --cpus 2 \
  -e JAVA_OPTS="-XX:+UseContainerSupport -XX:MaxRAMPercentage=60.0 \
  -XX:+ExitOnOutOfMemoryError -XX:MaxMetaspaceSize=256m -XX:MaxDirectMemorySize=128m" \
  registry.example.com/app:1.0.0

# 8. Verify new heap and RSS stay inside the cgroup
docker exec app jcmd <pid> VM.native_memory summary
docker stats app --no-stream
```

## Root Cause

- **Explicit `-Xmx4g` overriding container-aware default:** the app shipped with hardcoded heap that ignores cgroup limits (JDK 8u191+ defaults `MaxRAMPercentage=25%` *of the container limit* only when `-Xmx` is absent). A hard `-Xmx4g` forces the JVM to claim 4 GB heap regardless of the 2 GB cap.
- **Native memory not part of the budget:** metaspace (hundreds of MB for many classes), code cache (JIT), per-thread stacks (thousands of HTTP threads × `-Xss`), direct buffers/caches (esp. the serializer/Kafka/netty), GC structures. These add 0.5–1.5× heap on top — the "RSS 3.2 GB" the team quoted included them.
- **cgroup v1/v2 semantics confusion:** on cgroup v1, memory counted includes page cache attributable to the container unless reclaimed; reads/writes may push usage transiently over the cap and trigger the OOM killer even if "live working set" is under the cap. On v2, `memory.max` includes kernel pages & file cache the same way; the fix set must include `--memory-swap` policy (unlimited swap means the box survives swap-backed allocation but goes slow — a different failure mode).
- **`-Xms = -Xmx` committment at startup:** `-Xms4g` *commits* the 4 GB up front (it's in RSS by boot time, before your code runs). This is the *exact* reason you see `mmap failed` instantly: the JVM's startup reservation of 4 GB exceeds a 2 GB cap — nothing the app can do once it's running; the reserved space alone is over budget.
- **The "removed-limit" trap:** the app ran at 3.2 GB RSS without limits — that's exactly what it needs; the new 2 GB limit isn't "wrong" per se, it's *wrong for this footprint*. The engineering error is static (2 GB vs 3.2 GB). The correct choice is either a 4.5–5 GB cgroup (heap+overhead+headroom) or a 2 GB cgroup with a 1.2 GB heap contract, not "no limits."

## Immediate Mitigation

1. **If the app is critical:** raise the cgroup to a *measured* value while preserving cost guardrails — not unlimited, not the old 2 GB:
   ```bash
   docker update -m 5g --memory-swap 6g <container>
   ```
   Then watch `docker stats`; set it to the observed RSS + 25% headroom, for example 4 GB → `-m 4.5g`.
2. **Inject a consistent JVM right-sizing override, no image rebuild:**
   ```bash
   docker update <container> --env JAVA_OPTS="-Xmx3g -XX:MaxMetaspaceSize=256m -XX:ReservedCodeCacheSize=128m -XX:+ExitOnOutOfMemoryError"
   ```
   (You may need to recreate the container to apply env changes; that's fine as long as it's drained by the LB first.)
3. **Switch off the failing pattern**: change the Dockerfile start line so local double-default doesn't fight the shared cap:
   ```dockerfile
   # old: ENTRYPOINT ["java", "-Xms4g", "-Xmx4g", "-jar", "app.jar"]
   # new: no -Xms/-Xmx in the image; env-only:
   ENTRYPOINT ["sh", "-c", "exec java $JAVA_OPTS -jar /app/app.jar"]
   ```
4. **If the app truly needs 3.2 GB+ steady RSS, keep the 4 GB heap but grant a matching cap** (`--memory=6g`), and verify with `docker stats` that RSS stays under cap +5% for a soak window.
5. **Never panic into `--oom-kill-disable`** — it weakens cgroup protection host-wide and hides the mis-sizing; it's a mislabel, not a fix.

## Permanent Fix

1. **Adopt the ratio contract, documented for every service:**
   ```
   JavaHeapMax = 60% of ContainerLimit
   ContainerLimit = (Heap + native overhead) × 1.25 (headroom)
   ```
   Encode as `-XX:MaxRAMPercentage=60.0 -XX:InitialRAMPercentage=50.0` (no hard `-Xmx` in the image).
2. **Make the image limit-agnostic**: strip `-Xms/-Xmx` from the image; inject only via env (`JAVA_OPTS`) at the orchestrator/deployment level so each env (dev/staging/prod) sizes correctly.
3. **Set native-memory caps explicitly** so the "overhead" is a number you own, not a surprise:
   ```
   -XX:MaxMetaspaceSize=256m
   -XX:ReservedCodeCacheSize=128m
   -XX:MaxDirectMemorySize=128m (if NIO buffers are bounded)
   -XX:ActiveProcessorCount=2    (align thread pool with --cpus)
   ```
4. **Enable NMT (NativeMemoryTracking) in prod** (`-XX:NativeMemoryTracking=summary`) and refresh the summary via `jcmd` — the resulting "native vs heap" line automatically keeps the overhead estimate current across upgrades.
5. **Gate it in CI**: a lint that fails images containing `-Xmx` greater than the deployment manifest's limit, and a golden-file contract per service (limits in K8s `resources.requests/limits` or docker `-m`) reviewed in the same PR as the JVM flags.
6. **Do the OOM runbook drill once a quarter** so any on-call can execute the 3-line diagnosis (inspect OOMKilled → dmesg → NMT) and the two-option fix (shrink JVM vs grow cap) without improvising.
7. **Add heap-capacity alerting before runtime OOM** — live heap % (from Micrometer/JMX) > 80% for 10 min → warn, > 90% → page, because with container-aware sizing crashes become predictable.

## Monitoring

- **cgroup-level:**
  - `container_memory_usage_bytes / container_memory_limit_bytes` — the core ratio; alert at 85% sustained, page at 95%.
  - cgroup v2 `memory.events.oom_kill` as a synthetic counter → page on the *first* kill, not the tenth.
  - `docker stats` streamed to Prometheus (cadvisor) for per-container RSS + limit over time.
- **JVM-level (if JMX/Micrometer exposed):**
  - `jvm_memory_used_bytes{area=heap}` , `jvm_memory_max_bytes` — heap headroom.
  - `jvm_gc_live_data_size_bytes` (leak detector vs fixed heap).
  - Native memory if NMT exported: `jvm_nmt_*` probe or periodic `jcmd` capture → e.g., metaspace & code cache growth trend.
- **Process-level:**
  - Fatal error log capture for `hs_err_pid*.log` shipping (JVM prints them on crash); plus agent to collect recent logs pageable.
- **Alert semantics:**
  - "memory pressure" warn → (slack) → "OOM risk" page with `limit`, `rss`, `jvm_heap_used` — the on-call can pick shrink vs grow from the numbers.
- **Trend dashboards:** memory envelope (limit line, RSS line, heap line) per service per release — the visual that makes the "we removed limits" discussion a chart, not an argument.

## Security

- **Don't disable OOM protection in response to pressure** — a host OOM from a single mis-sized container kills co-tenants and resets their connections (availability incident ≥ the original one). Kernel OOM of the container is the *designed* backstop; removing it is removing a control, not fixing a bug.
- **Cgroup limits are also your multi-tenancy isolation**: on shared nodes, an unbounded JVM (heap + native) can starve neighboring tenants; per-tenant caps are a security boundary as much as a cost control.
- **Capturing the breach if a malicious thread spikes:** without a cap, attacker code can drive a memory-exhaustion DoS *inside* the container to the node; a hard cgroup is the blast-radius limiter.
- **Guard the diagnostics channels:** `jcmd`/NMT and JFR are powerful — don't expose JMX (`-Dcom.sun.management.jmxremote` bound to `0.0.0.0`) to the world; bind internally and authenticate, since the same channel that reads memory also writes.
- **Change control:** an ops engineer relaxing a prod limit to fix an OOM should log as a config change (trigger a review), since "memory limit changed" is a security-control documentation record.
- **SBOM/binaries:** if rebuilding with a smaller JVM (jlink/GraalVM) to fit limits, keep the SBOM updated — trimmed runtimes look "smaller" but their dropped modules change the vulnerability surface list.

## Production Considerations

- **Reliability:** predictable caps → predictable failure modes (fails fast with `ExitOnOutOfMemoryError` rather than wedging and silently degrading latency). A bound JVM under a bound cgroup is far more deterministic than an unbounded one.
- **Scalability:** with a fixed per-pod envelope, horizontal autoscaling computes correctly — each replica claims `limit`, so a 2 GB limit × 10 replicas = 20 GB — the scheduler lists pods correctly; unbounded pods make the scheduler's arithmetic meaningless.
- **Cost:** right-sizing pods to *measured* footprints directly reduces your bill (a 5 GB pod that uses 2.5 GB is wasted spend) and improves bin-packing density on nodes.
- **HA:** consistent limits make failover predictions tractable — if every replica can host the footprint, node eviction or restart fits within the reservation instantly.
- **Compliance:** memory limit as a documented "contract per service" makes capacity reviews (SOC2/HIPAA audit: "did you budget for three months of growth?") answerable with one dashboard and one ticket trail.
- **Operational:** every OOM now has a *decision tree*, not a debate (shrink JVM vs grow cap), cutting MTTR; NMT + metrics convert "mysterious 137" into "heap 1.9 GB / metaspace 320 MB in a 2 GB cap — the fix is ..." minutes after the page.
- **Error budget angle:** incidents caused by wrong sizing are "self-inflicted" — a quarterly review of the envelope SLO (the app's memory never exceeding its cap) keeps the quality queue ahead of the outage queue.

## Senior-Level Answer

"You're not tuning Docker, you're reconciling *two contracts*: the cgroup limit and the JVM memory claim. `-Xmx4g` under `--memory=2g` is contradictory — the JVM will pre-commit its heap at startup (`-Xms4g`), so it's killed by its own reservation before your first request, or SIGKILLed at runtime the moment used memory crosses the cap. The correct model is a budget: total-container = heap + native (metaspace, code cache, thread stacks, direct buffers, GC) + headroom. Two sound configurations: (A) keep the 2 GB cap and size the JVM inside it with `-XX:MaxRAMPercentage=60.0` → heap ≈ 1.2 GB, or (B) honor the 4 GB heap and set the cap to ~7 GB. I default to (A) for cost isolation, and I bake the contract into artifacts: no hard `-Xmx` in the image, `JAVA_OPTS` injected per environment, native-memory caps set explicitly, NMT enabled, CI linter that rejects a `-Xmx` above the manifest limit, and alerts on heap/RSS pressure long before the cgroup OOM page. 'Remove the limits' is the trap answer — that just trades a single container failure for a host-wide one and signals you never actually understood the JVM's memory map."

## Architect-Level Answer

"I treat memory configuration as a *contract between the deployment platform and the runtime*, documented and machine-verified, not a per-service guess. The platform defines the accounting model: every workload declares a resource envelope (`requests/limits` or `-m/-cpus`), and the runtime contract is that the JVM derives its own sizing from that envelope — hence container-aware defaults (`-XX:MaxRAMPercentage`), explicit native caps, and the golden rule that *heap ≤ ~60% of the cgroup*. Hardcoded `-Xmx` values (or, worse, images that assume unlimited memory) are treated as design errors and caught in CI, because once images claim memory outside their declared envelope, autoscheduling, bin-packing, and node-failover math all silently break.

Strategically this is one instance of a broader platform rule: *the code must not be able to outspend its execution slot*, for a JVM or any runtime. I'd standardize per-stack golden images that encode these contracts, add capability for NMT export to pull native-memory into the same dashboards as heap, and run a quarterly 'envelope audit' where every service's actual RSS is compared with its declared cap so drift (a dependency that quadruples direct buffers) surfaces as a ticket, not a 3 AM kill. Cost, reliability, and security all converge on the same principle: the smaller and more truthful the envelope, the more you can schedule, the fewer cross-tenant failures you cause, and the cheaper the fleet is — that's the business case for disciplined memory contracts."

## Follow-Up Questions

1. `-XX:MaxRAMPercentage=60.0` was set, but the JVM still uses 1.9 GB heap in a 2 GB container at peak. Explain precisely what `MaxRAMPercentage` computes, when the percentage is ignored (explicit `-Xmx`), and what the *other* sources of the 60% limit are (cgroup v1 `memory.limit_in_bytes` vs v2 `memory.max`, and the JVM's `-XX:MaxRAM` fallback).
2. A JVM with `-Xmx2g` inside `--memory=3g --memory-swap=4g` runs for a day, then OOMs. Compare: where does swap enter the story on cgroup v1 vs v2, and what does setting `--memory-swap=-1` (unlimited swap) actually change in the OOM outcome? Is "it ran without limits" ever a fair comparison when swap was unlimited before?
3. You see `ExitCode=1`, not 137, plus `Native memory allocation (mmap) failed to map...` in the logs — *before* the kernel OOM killer runs. Reconstruct the startup sequence and explain why the JVM gives up on its own here, and why you can't "fix" it by adding more threads or rebooting; what flag change fixes it *and why*?
4. The app's RSS is 3.2 GB with no direct-buffer code, and your budget says heap 4 GB + 1 GB native — but the container OOMs at 5.5 GB. Which cgroup/cpu/scheduling detail could explain RSS under the *reported* cap being killed (think: page cache, memory.threshold!, NUMA, memory.high soft limits), and how do you prove it from `/sys/fs/cgroup` and `dmesg`?
5. You're asked to design the "memory policy" for a JVM fleet: three service classes (`web-tier 200ms p99`, `batch-cron`, `etl-stream`). Propose the limit + JVM-flag matrix for each, list the monitoring signals each class needs, and explain what a shared "JVM golden base image" looks like (env overrides only) so that *no* future developer can write `-Xmx` into the image again.