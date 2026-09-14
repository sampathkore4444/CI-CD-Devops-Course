# 037. Docker Container Works on Dev Laptop but Crashes in Production

## Scenario

A developer's Dockerized application runs perfectly on their MacBook (M1 chip, 16GB RAM). Every test passes locally, and the image builds cleanly. After promotion through CI/CD and deployment to the production Linux servers (Ubuntu 22.04, amd64 architecture), the container starts but immediately crashes a few seconds later.

Docker logs show:

```
docker logs --tail 100 <container_id>
```

Output:

```
Native memory allocation (mmap) failed to map 268435456 bytes for committing reserved memory.
#  There is insufficient memory for the Java Runtime Environment to continue.
# Native memory allocation (mmap) failed to map 268435456 bytes for committing reserved memory.
```

`docker inspect` shows `"OOMKilled": true` and exit code 137. The developer insists "it works on my machine" and refuses to believe the code is the problem. The production cluster has plenty of RAM (the host has 64GB), so memory exhaustion seems impossible. The on-call engineer is paged at 3:00 AM during a critical sales window.

## Interviewer Question

"A developer's Dockerized application runs perfectly on their Mac laptop, but when it's deployed to production Linux servers, the container starts and immediately gets killed with exit code 137 (OOM Killed). The host has 64GB of RAM and 4GB is available. The developer insists 'it works on my machine.' As the DevOps engineer on call, walk me through exactly how you would investigate this, what the likely root causes are, and how you would fix it permanently."

## What I Should Think About

- Exit code 137 = 128 + 9 (SIGKILL), typically an OOM kill by the kernel or cgroup limit
- "Works on my machine" is almost always an environment difference:
  - **Architecture mismatch**: Mac (arm64/M1) vs production (amd64) — naive or native libraries, syscall differences
  - **Resource limits**: Dev laptop has no `--memory` limit; production has cgroup limits
  - **JVM/GC heap sizing**: JVM sizes heap to container memory if flags are set, or uses host memory if not
  - **Default `-Xms`/`-Xmx`**: Java defaults may be based on host total RAM, not container cgroup
- Key diagnostic steps: check full exit code, `docker inspect` OOMKilled flag, check container memory limits, check host dmesg for oom-killer events
- The difference between cgroup OOM and host OOM is critical — `docker inspect` tells you which
- Verify how the image was built: multi-arch or single arch? `docker buildx` vs plain `docker build`
- Do NOT redeploy blindly. Reproduce, isolate variables, one change at a time
- Never dismiss the developer; reproduce facts first, then show evidence

## Ideal Answer

"As the on-call DevOps engineer, I would treat this as a P1 incident requiring immediate triage. I would start by collecting the ground truth from production rather than debating with the developer.

**Step 1 — Gather evidence.** I'd run `docker inspect <container>` and check the `State.OOMKilled` field and the `HostConfig.Memory` setting. An empty `HostConfig.Memory` with `OOMKilled: true` points to a host-level OOM killer or an architecture-native crash (JVM native mmap failure), while a set limit with OOMKilled means the cgroup memory limit was hit. I'd also check `docker logs` for the actual JVM `Error occurred during initialization of VM` messages and `dmesg | grep -i oom` on the host.

**Step 2 — Check the architecture mismatch.** The most common culprit in the Mac-vs-Linux story is ARM vs AMD64. On an M1 Mac, `docker build` produces `linux/arm64` images. If the production registry/CI pipeline re-tags that same image onto amd64 servers, every container will crash on startup — JVM inside a Rosetta/qemu emulated environment often fails with exactly the native mmap errors we see. The fix is multi-architecture builds with `docker buildx build --platform linux/amd64,linux/arm64`.

**Step 3 — Check the JVM resource control.** JVMs do NOT automatically respect cgroup memory limits on older JDK releases. A JVM with default `-Xmx` on a 64GB host may reserve 25% of host RAM (16GB). If the container has a 2GB memory limit, the JVM reserves memory beyond the cgroup limit, and the kernel OOM killer terminates the container. On modern JDK 8u191+/11+, `UseContainerSupport` is on by default, but if the Dockerfile starts a Spring Boot app via a plain `java -jar` without `-XX:MaxRAMPercentage`, sizing can still go wrong with older toolchains.

**Step 4 — Reproduce locally, then fix.** I'd verify the image's platforms by inspecting the manifest, run the image locally with `docker run --memory=2g --platform linux/amd64`, and converge on a single-variable fix: pin JVM heap via `JAVA_OPTS="-Xmx1024m -XX:MaxRAMPercentage=60.0"` or use jlink for a minimal JVM.

The permanent fix combines: multi-arch builds pinned to runtime platforms, explicit `-XX:MaxRAMPercentage`/`-Xmx` matching the container limit, running all artifacts through the same CI pipeline (never hand-built images), and alerting on `OOMKilled` events immediately."

## Architecture

```
DEV (M1 MacBook)                          PROD (Linux x86_64)
┌─────────────────────────┐              ┌──────────────────────────────────┐
│ docker build (arm64)    │              │ Docker Host (64GB RAM)           │
│ container runs natively │  image       │ ┌────────────────────────────┐   │
│ JVM reserves small heap │ ───────────► │ │ Container --memory=2g      │   │
│ cgroup default: none    │              │ │ JVM reserves ~16GB (25%    │   │
│ → no OOM                │              │ │ of host, ignores cgroup)   │   │
└─────────────────────────┘              │ │ → mmap fails → OOMKilled   │   │
                                         │ │ → exit 137                 │   │
                                         │ └────────────────────────────┘   │
                                         └──────────────────────────────────┘
    Root causes:
    1. arch mismatch arm64 image on amd64 host (Rosetta/qemu)
    2. JVM heap ignores cgroup limit (no MaxRAMPercentage)
    3. base image not multi-arch / untagged platform drift

    Desired state:
    ┌─────────────────────────┐
    │ docker run -m 2g \      │
    │   -e JAVA_OPTS="-Xmx1500m" │
    │ buildx --platform amd64,arm64 │
    │ → JVM heap < cgroup limit → stable │
    └─────────────────────────┘
```

## Investigation

1. **Confirm the exit code and OOM state.** Run `docker inspect <container_id> --format '{{.State.ExitCode}} {{.State.OOMKilled}}'`. Exit code 137 with `OOMKilled: true` confirms kernel/cgroup OOM termination. Exit code 137 with `OOMKilled: false` usually means an external SIGKILL or an explicit `kill -9`.

2. **Read the application logs.** `docker logs <container_id> 2>&1 | tail -100`. Look for JVM init failures, native mmap errors, or the string `There is insufficient memory for the Java Runtime Environment to continue`.

3. **Check the host OOM killer.** On the host run `dmesg | tail -50` or `journalctl -k | grep -i oom`. Search for `Out of memory: Killed process` and note the process name and whether the OOM was triggered by cgroup or host pressure.

4. **Inspect resource limits.** `docker inspect <container_id> --format '{{.HostConfig.Memory}} {{.HostConfig.NanoCpus}}'`. Compare the container limit against what the app actually needs. Also check the stack `docker service` or orchestrator (K8s) limits.

5. **Check the image architecture.** `docker image inspect <image> --format '{{.Architecture}} {{.Os}}'` and compare with `docker inspect <host_container> --format '{{.Architecture}}'` or simply `docker info`. If the image says `arm64` and the host is `amd64`, that's a smoking gun.

6. **Inspect the JVM settings.** Look at the Dockerfile ENTRYPOINT/CMD. If it runs `java -jar app.jar` with no `-Xmx`/`MaxRAMPercentage`, the JVM uses host-derived defaults. Calculate: default Ergonomics heap = 1/4 of host RAM.

7. **Reproduce with matched constraints.** On a Linux box run `docker run --rm --memory=2g --cpus=4 --platform linux/amd64 <image>` and observe if it crashes identically. Then run without the memory limit to confirm the limit is the trigger.

8. **Compare dev vs production image.** `docker image inspect` both and diff the `Architecture`, `Env`, and `Entrypoint`. If they differ, the CI pipeline is re-tagging/wrong-platform-submitting.

## Commands

```bash
# 1. Get full failure state of the crashed container
docker inspect <container_id> \
  --format 'ExitCode={{.State.ExitCode}} OOMKilled={{.State.OOMKilled}}'

# 2. View last 100 lines of logs
docker logs --tail 100 <container_id> 2>&1

# 3. Check host kernel OOM events
sudo dmesg | tail -50 | grep -i -E "oom|out of memory|killed process"

# 4. Inspect the resource limits actually applied
docker inspect <container_id> \
  --format 'Memory={{.HostConfig.Memory}} MemorySwap={{.HostConfig.MemorySwap}} CPUs={{.HostConfig.NanoCpus}}'

# 5. Verify image and host architecture
docker image inspect <image> --format 'Image Arch/OS: {{.Architecture}}/{{.Os}}'
docker inspect <container_id> --format 'Container Arch/OS: {{.Architecture}}/{{.Os}}'

# 6. Reproduce with a memory limit on a Linux host
docker run --rm --memory=2g --cpus=2 --platform linux/amd64 <image>

# 7. Build a proper multi-arch image
docker buildx create --name multicore --use
docker buildx build --platform linux/amd64,linux/arm64 \
  -t registry.example.com/app:1.0.0 --push .

# 8. Run the fixed container with explicit heap and limit
docker run -d --name app \
  -m 2g \
  -e JAVA_OPTS="-XX:MaxRAMPercentage=60.0 -XX:MaxMetaspaceSize=256m" \
  registry.example.com/app:1.0.0
```

## Root Cause

- **Architecture mismatch (most common Dev/Mac vs Prod/Linux):** M1 Mac builds create `linux/arm64` images. Deployed to amd64 hosts, the runtime may emulate (Rosetta/qemu) or simply fail. Native code in the JVM or any compiled library returns mmap/mprotect errors instantly.
  - *Eliminate:* `docker image inspect <image> --format '{{.Architecture}}'` on the deployed tag. It must match the host. If it says `arm64`, this is it.
- **JVM ignores cgroup memory limits:** Pre-JDK 8u191 / without `UseContainerSupport`, `-Xmx` defaults to 25% of *host* RAM. A 2GB container limit on a 64GB host causes an immediate mmap failure — JVM reserved more than the cgroup allows.
  - *Eliminate:* Inspect the JVM args in the image. If there's no `-Xmx`/`-XX:MaxRAMPercentage` and `UseContainerSupport` is absent, this is the cause. Fix: add `-XX:MaxRAMPercentage=60.0` or terminate with `-Xmx1500m`.
- **Heap + metaspace + thread stacks exceed the cgroup limit:** Even with container support, fixed overhead (metaspace, code cache, direct buffers, native threads) can push RSS over the limit when heap sizing is set near the whole limit.
  - *Eliminate:* Run `docker stats` and watch `MEM USAGE` right before the crash. Compare to the limit. Leave headroom: heap + ~700MB overhead.
- **Host-level memory pressure (other tenants):** If the cgroup limit is unset (`HostConfig.Memory = 0`), the host OOM killer may pick this container when the host runs out of memory.
  - *Eliminate:* `docker inspect` shows `Memory=0`; `dmesg` shows host OOM events; check other containers/host processes for memory hog.
- **Wrong base image / platform drift through CI:** The pipeline might rebuild with a different base tag, or the same tag is overwritten by a multi-stage build for the wrong platform.
  - *Eliminate:* Diff `docker image history` between dev-built and CI-built images; check `FROM` line and `--platform`.

## Immediate Mitigation

1. **If the app is critical:** Revert to the last known-good image tag or the last release that ran stably in production (`docker service update --image <prev-version>` or rollback in orchestrator).
2. **Temporary band-aid while root cause is investigated:** Raise the container memory limit to give the JVM more room (`docker update -m 4g <container>`), or set `-XX:MaxRAMPercentage` via env override in the orchestrator:
   ```bash
   docker run -d -m 3g -e JAVA_OPTS="-XX:MaxRAMPercentage=70.0" <image>
   ```
3. **If architecture is the cause** — you cannot run arm64 natively on amd64 reliably. Rebuild the image on the pipeline with `--platform linux/amd64` or push the correct multi-arch manifest and pull with the right platform tag:
   ```bash
   docker buildx build --platform linux/amd64 -t app:fix --push .
   ```
4. **Restart with explicit limits** after the change and verify with `docker stats` that memory usage plateaus below the limit instead of climbing to it.

Security note for mitigation: never disable `--memory` limits "temporarily" to keep the container up — that trades a crash for a host-wide reboot when the container leaks. Scope the mitigation to a single container and time-bound it to the root-cause window.

## Permanent Fix

1. **Multi-arch builds through CI only.** Disable all hand-built "works on my machine" images. Use `docker buildx build --platform linux/amd64,linux/arm64 --push` in the pipeline so the manifest always matches the target fleet. Verify architecture in CI:
   ```bash
   for tag in linux/amd64 linux/arm64; do
     docker buildx imagetools inspect registry/app:1.0.0 --raw | grep -q "$tag"
   done
   ```
2. **Codify JVM/container memory rules.** Standardize a base image with sane defaults:
   ```dockerfile
   FROM eclipse-temurin:21-jre
   ENV JAVA_OPTS="-XX:MaxRAMPercentage=60.0 -XX:MaxMetaspaceSize=256m -Xss512k"
   ENTRYPOINT ["sh", "-c", "exec java $JAVA_OPTS -jar /app/app.jar"]
   ```
   Always set `-XX:MaxRAMPercentage` < 100 and reserve headroom for native memory.
3. **Set resource limits at the orchestration layer** and make them a first-class review item in every deployment pipeline (K8s `resources.requests/limits`, Docker `--memory`, `--memory-swap`, `--cpus`). Never leave untracked limits.
4. **Treat the container limit as a contract with the JVM.** Document "2GB limit + 60% heap" as the standard. Add a startup smoke test that fails CI when heap settings conflict with limits.
5. **Add CNI/registry-level platform gating** so a `linux/arm64` manifest can never be deployed to an amd64 fleet.
6. **Deprecate dev-built images entirely:** require every image to come from CI with a build id in the label (`org.opencontainers.image.revision`).

## Monitoring

- **Immediate alerts:**
  - `container_process_oom_killed` or `docker_container_exit_code{code="137"}` → page immediately (Prometheus `cadvisor` / `node_exporter` metrics).
  - `container_memory_usage_bytes / container_memory_limit_bytes > 0.85` for 5 min → warning; `> 0.97` → page.
- **Trending metrics:**
  - RSS headroom: `limit - usage` per container over 7/30 days to size limits correctly.
  - `jvm_heap_used` vs `jvm_max_heap` (if Micrometer/Prometheus JVM agent present) to catch leaks before OOM.
  - Container restarts this week per service — spike detection.
- **Logs to ship:**
  - JVM fatal error logs (`hs_err_pid*.log`), `journald -u docker` OOM/Kill lines, `dmesg` filtered to `oom`.
- **Dashboards:**
  - Per-service exit code distribution over time (135/137/143) — a flat 137 line means a recurring limit tuning issue.
- **Synthetic check + observability note:** Keep the mean-time-to-restore visible: alert on "restarted in last 5 minutes" for prod workloads, because restarts cluster around deploys.

## Security

- **Architecture as a compliance issue:** Running untested emulated architecture (qemu/arm64-on-amd64) introduces undefined behavior in syscall backdoors in native code — a supply-chain and availability risk. Gate images by platform in the registry and CI.
- **Never disable OOM protection to "fix" memory issues:** An unconstrained container can kill the host (node OOM) taking down all tenants. Killing one container is far safer than a host reboot.
- **Pin base images and scan before promote:** The emergency fix must not pull a *newer* base image skipping `docker scan`/Trivy for CVEs. Rebuild from a pinned, scanned digest.
- **Verify unsigned/mutated images:** when dumping old images apply `docker trust` / Cosign sigstore verification so a "revert to last good" isn't a supply-chain poisoned one.
- **JVM secrets in env:** The resolution (`JAVA_OPTS`) should not log secrets when exposing the change to logs/dashboards.

## Production Considerations

- **HA / rollback:** Keep the last two good image digests pinned and pre-pulled on all nodes. A bad deploy must roll back to a digest, not a re-tag.
- **Scalability:** Horizontal scaling relies on consistent memory envelope. If containers are sized by autoscaler, the JVM heap ratio must scale with it — `MaxRAMPercentage` keeps this uniform.
- **Reliability / cost:** Oversized limits waste cluster capacity; undersized limits crash. Right-size via Percentile (p95/p99) memory traces — that reduces TCO while improving SLO.
- **Operational:** Standardize a `runbook` entry for "137 / OOMKilled in prod," so on-call engineers follow a defined path instead of improvising fixes in the middle of the night.
- **Compliance:** For regulated environments (SOC2, PCI), record why a limit or heap size was changed, and mark incidents that required mitigations. Error budget tracking helps justify adding capacity.
- **Cross-team:** The "works on my machine" gap is an organizational problem — CI parity (exactly the same image built and promoted) removes it permanently.

## Senior-Level Answer

"The instant I see 137/OOMKilled with mmap failures I don't fight the developer — I collect evidence. `docker inspect` for `OOMKilled` and `HostConfig.Memory`, `dmesg` for kernel context, and `docker image inspect` for the platform. The two classic culprits behind 'works on my Mac' are: (1) an arm64 image from an M-series Mac being pushed to amd64 prod — the CI is promoting a manifest that doesn't match the target platform, and (2) a JVM running without `UseContainerSupport` or `-XX:MaxRAMPercentage`, so it sizes its heap from the 64GB host, not the 2GB cgroup, and gets SIGKILLed at startup. I fix it permanently by enforcing multi-arch `buildx` builds only through CI, moving `-XX:MaxRAMPercentage=60.0` into the base image env, making memory limits a review gate, and wiring `OOMKilled`/exit-137 alerting into the page path. And I always reopen the incident to see if we were running emulated architecture, which deserves a CVE-grade follow-up."

## Architect-Level Answer

"At the architecture level this incident is a signal of two systemic failures, not one bug. First, artifact governance: the pipeline promoted a platform-blind image; any artifact that can exist in dev but not run in prod contradicts the definition of a deployment artifact. I would model the pipeline as 'one manifest, validated platforms,' where every tagged image is a multi-arch manifest (`linux/amd64,linux/arm64`) built and signed in CI, cleansed of CVEs, and promoted by digest, never by tag. Second, resource contracts: containers need a declared envelope (CPU/memory) enforced at runtime and communicated to the runtime in code. For JVM workloads that means container-aware defaults (`-XX:MaxRAMPercentage`, `-XX:ActiveProcessorCount`) baked into the platform base image, validated by a startup self-test so developer repros fail in CI, not prod.

From a business standpoint I would treat 'works on my machine' as a queue that must be driven to zero by eliminating hand-built images and personal registries. The platform team owns the golden base image, the pipeline owns platform validation, and the release process owns digest-based promotion. On the operational side I'd unify on signals — restart rate, OOMKilled count, memory headroom, and exit-code distribution — feeding an SLO on '0 deployments with artifact/runtime mismatch.' That turns a 3 AM page into a one-line runbook check and, more importantly, removes an entire class of incidents."

## Follow-Up Questions

1. You open the container's config and see `OOMKilled: false` but the exit code is still 137. Your partner says "it's definitely OOM." What would you check next to prove the actual cause — and what other conditions produce 137 besides OOM?
2. The JVM has `-XX:MaxRAMPercentage=60.0`, headroom looks fine, but the container still gets OOM-killed after 3 days of uptime. Walk through everything you'd investigate (hint: think metaspace, thread stacks, native memory, and `docker stats` trends).
3. Your developer says the app also runs in Docker Desktop on Windows with WSL2. How does WSL2's memory model make the "works on my machine" story even more misleading than a Mac?
4. Explain `docker update -m`, `--memory-swap`, `--oom-kill-disable`, and `--memory-reservation`: what does each do, and why is `--oom-kill-disable` dangerous for a JVM app with a fixed heap?
5. You discover the CI is building for the wrong platform because a previous engineer set `BUILDPLATFORM` incorrectly in the `buildx` invocation. Describe how you'd design a *platform safety gate* in CI that makes this class of mistake impossible to ship again.