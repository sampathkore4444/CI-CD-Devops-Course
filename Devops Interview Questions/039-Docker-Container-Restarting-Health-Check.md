# 039. Docker Container Restarting - Health Check Failing

## Scenario

A production API container (Node.js + Express, or equivalent, e.g., a REST service) is entering a **restart loop**. `docker ps` shows:

```
CONTAINER ID   IMAGE         COMMAND                  STATUS
a1b2c3d4e5f6  api:1.4.0    "node src/server.js"     Up 25 seconds (health: starting)
```

Then it restarts. Every ~30 seconds the cycle repeats:

```
NFO  Server started on port 8080
INFO  Connected to database
WARN  no response from /healthz within 2s, hitting http timeout
INFO  Server started on port 8080
INFO  Connected to database
```

The Docker healthcheck is:

```yaml
healthcheck:
  test: ["CMD", "/usr/bin/wget", "-qO-", "http://localhost:8080/healthz"]
  interval: 30s
  timeout: 5s
  retries: 3
  start_period: 10s
```

Exit code from `docker inspect` is always 1 (the entrypoint process exits) — but the healthcheck reports `unhealthy` before the restart. The app **starts** successfully (logs: "Server started on port 8080") and *then becomes unresponsive after ~25 seconds*. The service is behind a load balancer that is now returning 502s to customers.

The team updated the code yesterday (a new in-memory cache was introduced) and the incident started 12 minutes after that deploy.

## Interviewer Question

"A container is stuck in a restart loop — it starts, healthcheck fails, it restarts every ~30 seconds. Logs show the app 'Server started,' then 25 seconds later it becomes unresponsive. The load balancer is dropping to 502s. Show me exactly how you investigate the restart loop, how you distinguish 'the healthcheck is wrong' from 'the app is actually broken,' and what you'd do in the first 10 minutes versus the permanent fix."

## What I Should Think About

- **Separation of concerns in healthchecks**: healthchecks probe *readiness* vs *liveness*. A failing liveness check means "process is alive but wedged / not serving correctly"; a failing readiness probe should *remove from LB*, not kill. Dockers healthcheck conflates these — know when a restart is the *wrong response* to a failure.
- **Look at the exit reason first.** `docker inspect` `State.ExitCode` + `State.OOMKilled` + `State.Error`. Then look for the code path that exits: explicit `process.exit(1)`, unhandled exception, or the healthcheck being the CI/CD's failure.
- **Healthcheck false negatives vs true negatives:**
  - Endpoint exists? Path correct? Port correct? Is the probe running inside the container's processor (wget) vs the app actually listening?
  - The probe hitting an **HTTP 4xx** (e.g., 401 requiring auth, or wrong path) returns non-zero → marks unhealthy even though the app is fine.
  - `start_period` too short: app takes longer than seconds to warm up → every start is killed by its own healthcheck.
- **Why would the app be responsive at t=0 and dead at t=25s?** Common: **a CPU/memory spike from an in-memory cache** pinned during startup (loading the dataset), thread explosion, event loop blocked (deadlock/blocking sync call), or a **node/JS event-loop-blocking `while` after startup**. Logs "started" before initialization completes.
- **Three clocks colliding**: `interval`, `timeout`, `retries`. If interval+timeout×(retries) < time-to-warm, every container is killed before it ever passes. Overlap: the "starting" phase with `start_period`.
- **The restart loop can hide the real failure**: the 502s through the load balancer mean the LB's own healthcheck (different path/port) is failing too — this is the "true unhealthy" signal.
- **First-10-minutes rules:** don't restart blindly; capture `docker inspect`, `docker logs --since`, `docker stats`, and look at the last few seconds of the container lifetime; then get a dump of what the process was doing (in Node: check for `--prof`/`--heapsnapshot`; Java: **jstack**, **jmap -dump:live,format=b**).

## Ideal Answer

"I treat this as an immediate production incident and split my response into triage and root cause.

**First 10 minutes (restore service):**
1. I read `docker inspect <container> --format '{{.State.ExitCode}} {{.State.OOMKilled}} {{.State.Error}}'` and `docker logs --since 10m <container> 2>&1 | tail -80`. The exit code and the *final* log lines tell me whether the process is exiting itself (app-caused) or being killed (OOM/healthcheck policy). If it's an unhealthy-then-restart, I want the LB to *drain* before restart, so I scale the deployment down or disable the replica in the LB so we don't pour 502s on customers.
2. If the app is the thing exiting, I disable *that image's* auto-restart quickly: for Docker, replace `restart: always` with a manual run; for K8s, I temporarily modify the livenessProbe `failureThreshold`. But I never lower the probe just to silence the alert — I want to stop the crash loop, not the signal.
3. I check `docker stats` during the window it stays up (25s) — CPU and memory trajectory tells me if it's a resource-bound swallow or a hang.

**Root cause (after service level restored):** The classic story here is the 25-second lull: 'Server started' at t=0, then silence. For the new in-memory cache the team added: startup onboards the cache (loads data), and if the load is a **synchronous** event-loop-blocking activity that takes > timeout, /healthz never answers during onboarding. But the interesting symptom is it becomes dead *after* 25s — meaning something other than the health probe shapes the app. I'd verify three hypotheses: (a) the healthcheck path/port is wrong (auth on /healthz, port in container differs), (b) the event loop or thread pool is fully wedged by the cache load / a `while` loop / a blocking DB query after startup, (c) the healthcheck `start_period` is too short for the app's real warmup. I test (a) with `docker exec` curl/wget inside the container; (b) with a thread dump at the 25s mark (Node: `kill -SIGUSR1` + `node --prof`, or `--inspect`; Java: `jstack <pid>` while inside the namespace); (c) by reading the healthcheck config and matching against a `docker cp` of the app's startup logs timing.

The permanent fix is a *split of liveness vs readiness*: readiness answers 'can I serve traffic?' — gateway/gRPC load balancer uses it; liveness answers 'should I be restarted?' — Docker/K8s uses it, with a *generous* `start_period` and low `failureThreshold`. And the app gets a real health endpoint that does a cheap internal self-test (DB ping, cache health) with a bounded timeout, not just 'HTTP 200.' For this specific cache: load it **lazily/asynchronously after listen**, so the server can answer health during warmup."

## Architecture

```
Deprecated states (Docker healthcheck = poor liveness+readiness conflation):

Docker:  ┌────────────────────────────────────────────┐
         │ container running → healthy → not-healing │
         │ healthcheck hit "waiting for something"   │
         │ → unhealthy (marks a backend down)        │
         └─────────── exits 1 → restart policy kicks ┘

What happens with a real "starts, then wedges" app:
 start_period (10s) ─── listen("Server started") ─── 25s later ─── bool → dead

// idealized timeline
t=0      listen on :8080 banner printed
t=0..25  in-memory cache synchronous load (CPU/blocked), /healthz unresponsive
t=25s    restarted by orchestrator (less than start_period+interval+timeout)

LB side:
c=0..25  LB probe (different path /actuator/health) → 000 / 502 → replica removed
         → but during autoscaled roll the load collapses → 502s for customers

Better architecture (K8s-style separation):
 readinessProbe: "/api/ready" → 200 only after cache+DB warm   (LB traffic gate)
 livenessProbe:  "/api/live"  → 200 as long as event loop /healthz answers
                 startPeriodSeconds: 40-60 (match true warm time)
                 timeoutSeconds: 3, failureThreshold: 3
Docker-only equivalent: replace CMD wget healthcheck with a small
supervisor that reports "ready" vs "live" on different files/endpoints.
```

## Investigation

1. **Establish the container state.** `docker ps -a`, then `docker inspect <id> --format '{{.State.ExitCode}} {{.State.OOMKilled}} {{.State.Error}} {{.State.StartedAt}} {{.State.FinishedAt}}'`. Note the exact duration of each run — 25s?
2. **Read the tail of logs.** `docker logs --tail 120 <id> 2>&1`. Look for the last actionable line before exit. Is there an exception/stack, or just "Server started" and then nothing?
3. **Look for the orchestrator's view.** If K8s: `kubectl describe pod/<pod>` — `Last State: Terminated, Reason: Error`, `Restart Count`, `Events` often show *Liveness probe failed* exactly. For Docker swarm: `docker service ps` / `docker inspect`.
4. **Probe from inside the container** — determine if the app is really unresponsive:
   ```bash
   docker exec <id> bash -c "curl -sv http://localhost:8080/healthz; echo exit=$?"
   docker exec <id> sh -c "wget -qO- http://localhost:8080/healthz; echo exit=$?"
   ```
   If the exec itself opens a shell fine but the HTTP request times out → the app listens but the event loop is dead (or a firewall/port alias is wrong).
5. **Confirm the port/path the probe actually hits vs the app's listener:**
   - `docker exec <id> sh -c "netstat -an | grep -E 'LISTEN|8080'"` or in a Node container `ss -ltnp`.
   - Compare to the healthcheck definition (`docker inspect --format '{{json .Config.Healthcheck}}' <id>`).
   - A `/healthz` behind an OAuth/API-key middleware returning 401 → `wget` treats `401` as an error exit code → false unhealthy.
6. **Observe resource trajectory during the 25s window**:
   - Open a monitor: `while true; do docker stats <id> --no-stream; sleep 2; done` — or `docker stats` in a loop. CPU spiking to 100% / memory ballooning is a smoking gun.
7. **Capture a thread/stack sample mid-wedge.** For Node: run the container with `--inspect` or send `kill -SIGUSR1 <pid>`; or `node --prof` the container command. For Java: `docker exec <id> jstack <pid> | head -80`, mind PID namespaces.
8. **Reproduce the 25s wedge offline:** `docker run -d -p 8080:8080 <image>` on an isolated host with the same env and watch — if it wedges there too with the same timing, it's app logic, not infra.
9. **Check `docker events` / daemon logs** to see how Docker decided to restart (restart policy vs healthcheck-result) — `docker events --filter container=<id>`.

## Commands

```bash
# 1. Why did it exit?
docker inspect <container_id> \
  --format '{{.State.ExitCode}} OOM={{.State.OOMKilled}} Err={{.State.Error}}'

# 2. Restart policy + healthcheck config
docker inspect <container_id> \
  --format 'restart={{.HostConfig.RestartPolicy.Name}} hc={{json .Config.Healthcheck}}'

# 3. Last logs before each death window
docker logs --tail 150 <container_id> 2>&1

# 4. Probe from inside (is the app actually serving?)
docker exec <container_id> sh -c "curl -sm 2 -o /dev/null -w '%{http_code}' http://localhost:8080/healthz"

# 5. Watch resource trend for the 25s window
for i in $(seq 1 20); do docker stats --no-stream <container_id>; sleep 2; done

# 6. Confirm listener + port mapping
docker exec <container_id> sh -c "ss -ltnp 2>/dev/null | grep -E '8080|LISTEN'"

# 7. Capture Java thread dump at the 25s mark
PID=$(docker exec <container_id> sh -c "printenv HOSTNAME; pgrep -f 'java' | head -1")
docker exec <container_id> jstack $PID > /tmp/top_threads.txt

# 8. Capture Node process state
# run the container with exports:
EXPORT="--expose 9229"; docker exec node_proc kill -USR1 <pid>  # opens inspector
# or start with: node --inspect=0.0.0.0:9229 --prof src/server.js

# 9. Watch the daemon's view of the restart loop
docker events --filter "container=<container_id>" --filter "type=container"
```

## Root Cause

- **Healthcheck false-negative (misconfigured endpoint or port):** the most common. wget to `http://localhost:8080/healthz` but app listens on `0.0.0.0:9090` inside, or the endpoint returns 401/302 → healthy app labeled unhealthy.
  - *Eliminate:* run the same probe at container network namespace with `curl -v`; see the actual HTTP code; compare netstat listener to probe target; check whether probe path requires auth.
- **`start_period` shorter than true warmup and cache-load window:** app is legitimately "starting" (onboarding cache, JIT, connecting) for >25s and the probe runs in it. The 25s collapse = boundary: `interval 30s` with `start_period 10s` hands out exactly one chance — passes at t=10, then fails at t=30 mid-warmup. Retries/semantics are strict (3 failures).
  - *Eliminate:* find when the app's /healthz first returns 200 while it's actually *ready* (timed `for` loop hitting /healthz from outside at 1s intervals). Bring `start_period` and/or `interval` ahead of the warmup time.
- **Event loop / thread-pool wedge after successful listen:** the in-memory cache does a huge synchronous load or one blocking call after announce—one thread eats the CPU; event loop never gets to answer http. Node: a CPU-bound `for` over 10M items; Java: a single thread doing unbounded DB `SELECT` returning 500k rows affecting GC pause, but still "listening".
  - *Eliminate:* thread dump at the 25s mark shows one thread at `count()`/`fetchAll()` with `RUNNABLE` and -Xss growth; or GC/JIT percentage in profiler output exploding.
- **process exits from its own logic:** explicit `process.exit(code 1)` from an unhandled `uncaughtException`; bad config from env; port already bound (the 25s is just scheduling — 2 instances collide).
  - *Eliminate:* `docker logs` tail reveals stack or `Error: EADDRINUSE :8080`. Check duplicate publishes/LB reusing ports.
- **Restart policy caused by health procedure, not app:** LB/Docker marks unhealthy and *scaled across* but a *healthcheck in the container* is what's restarting. If the app never actually dies but the policy restarts on unhealthy — config error, not code error.
- **New code path, or "unknown" until proven:** after those are eliminated, use heap/profile dump; the "cache introduced yesterday" is the strongest change-correlation — diff recent commits against resource/time-onboarding path.

## Immediate Mitigation

1. **Stop the 502 flood:** scale the replica count to *zero* or disable the failing target in the load balancer — a *known-failing* backend feeding 502 is worse than removing it.
2. **Pause the restart loop to keep forensics:** temporarily run the container with `--restart=no` (or `kubectl scale deploy --replicas=0` then one pod with `--restart=never`): the last live state is preserved and you aren't chasing log tails across 50 runs.
3. **Freeze the deployment:** roll back the just-deployed image (the cache change) to the previous known-good tag **on a canary**, and only promote rollback to prod if the canary passes health; keep current data path ruled out to protect user sessions.
4. **If the healthcheck is the wrong config** (probe path/port/auth) — and you have evidence — fix the healthcheck and redeploy: `docker stop`/`start` the container, confirm health flips green.
5. **Widen the healthcheck** temporarily to survive warmup: raise `start_period` to an *honest* value and `retries` to give breathing room — but only as a named temporary, and tracked as a follow-up item, not left in place.
6. **Notify the affected LB/connection pool**: free the chunked session/DB connection that the wedged app holds; DB connections from crashed pods often linger (pool hysteresis is a common *second-ary* root cause after a rollback).

All of these are "stop the assume-broken default" reversible, observable steps — every one can be rolled back cleanly if the evidence changes.

## Permanent Fix

1. **Split readiness and liveness properly.** If on Docker alone, replace the single healthcheck with a small **init/sidecar supervisor** that reports `/api/ready` separately from `/api/live`. If on K8s: implement `readinessProbe` and `livenessProbe` distinctly; liveness is *cheap* (check 200 + event loop), readiness is *real* (DB ping, cache warm).
2. **Make the health endpoint honest:** the app exports `/healthz` that does self-checks with a bounded timeout (3s) and returns 503 with the failing component when not ready — never 200 on stale-but-dead.
3. **Fix the warmup in the app**: load the in-memory cache asynchronously after `listen` (doesn't block the loop), or warm explicitly in a pre-deploy job so serving starts ready.
4. **Set sane defaults in code** (not just the declarative healthcheck):
   ```yaml
   healthcheck:
     test: ["CMD", "node", "healthcheck.mjs"]
     interval: 10s
     timeout: 3s
     retries: 3
     start_period: 60s   # honest warm time after measurement
   ```
5. **Standardize an on-call runbook** ("Container restart loop / healthcheck failing"): first 10 min paths, the forensics set (inspect/logs/stats/thread-dump steps), rollback decision tree, and "when to raise the probe vs when to fix the app."
6. **Add regression coverage:** a CI or staging test that runs the container for `N` minutes with real load and *asserts* the healthcheck stays `healthy` through a full warm cycle — the 25s gap gets caught long before prod.
7. **Track deployment diff discipline:** the cache change was supposed to be transparent — add an "environment diff" check (health timing before/after each release) or a canary gate comparing app metrics so silent startup regressions are caught at the gate.

## Monitoring

- **Container-level:**
  - `container_restarts` total vs rate (`rate(container_restarts[10m])`) — alert on crash-loop signature vs blips.
  - `container_health_status{health="unhealthy"}` via cadvisor/Docker daemon metrics → page at 1+ minute of persistent unhealthy.
- **Application-level:**
  - `/healthz` response **latency** histogram (e.g., Prometheus from middleware) — a steady-state of `1s` that suddenly trends 3s+ predicts the 25s window long before it faults.
  - Garbage → GC/inspect metrics (Node: event loop lag via `process._getActiveHandles` histogram; Java: `jvm_gc_pause_seconds`), because "frozen 25s" usually shows GC/event-loop stalls to correlate.
- **Tracing/red-team holidays:** open a synthetic every minute hitting `/healthz` + a real transaction through the LB; synthetic to the *airport/DB path* catches the "am I really healthy" class instantly (healthpage 200 but DB pooled dead).
- **Alert designs:**
  - `healthz_latency{p} > 2s for 5m` → warn; `container_restarts > 3 in 10m` → page for prod.
  - On K8s: watch `kube_pod_container_status_restarts_total` and LivenessProbeFailed event-rate → page with `pod_phase` context.
- **Supply the evidence trail**: ship restart-reason (`OOMKilled`, `ExitCode`, `state.Error`) as a label/facet in the app log pipeline so a restart-loop incident opens with *why*, not just *when*.

## Security

- **Healthz endpoints are a reconnaissance surface under load:** an exposed `/healthz` that reveals component states (DB hostnames, cache keys, pool counts) can tip off an attacker about topology. Keep readiness/liveness paths internal (`localhost` bound inside, LB uses separate probe) and *never* put them on the public port if avoidable.
- **Never remove auth on a health endpoint "just for the probe"**: probe creds should be a dedicated token; the wget-against-`/healthz` anti-pattern often tears a hole into internal services when you re-expose the port.
- **Restart-loop boxes can mask takeovers:** a container that comes up/down every 30s might be re-establishing network egress in a way that rotates through DNS/HTTP external states. In extreme cases hook `docker events` into the SIEM for container lifetime spikes, not just exit codes.
- **Do not leak bind/addr/creds in the health output** (message like `unable to reach postgres:5432 user=admin`).
- **Probe failures through the LB**: even a legit restart path leaves the LB retrying with a short timeout — ensure LB probes carry timeouts that prevent request doubling (retries × timeout logged as 2x request globally).

## Production Considerations

- **HA / orchestration behavior**: restart loops burn a node's CPU + restart budget and can interfere with neighbors (OOM of shared resource). Prefer single-restart-tolerant healthchecks; the *fleet* (replicas + readiness gate) is the HA answer, not restart resilience.
- **Scalability**: readiness must come *true* before the autoscaler marks a pod ready, else scale-out worsens the restart storm (pods scaling while half-baked). Includes `minReadySeconds`/`progressDeadlineSeconds` on K8s.
- **Cost**: repeated local `docker pull` of a 2.5GB image × N crash-restarts can hammer the registry (the previous incident); crash-looping also wastes CPU/cloud credits.
- **Reliability / error budget**: a 502 flood from a restarting pod drains SLO fast; the 25s window should drive an SLO on "container available within T seconds of deploy."
- **Compliance**: change control on healthcheck tuning is a *config change* — record before/after `interval`/`timeout`/`start_period` runs; audits want to see the incident review, not the firefight.
- **Ops ergonomics**: stop tuning the healthcheck by trial-and-error — measure real warmup time, document it, and *own* the healthcheck config at platform level so "one team's probe" can't silently reset prod reliability.

## Senior-Level Answer

"The first thing I check with a restart loop is *why the liveness probe failed* and *what the process was doing in the last 25 seconds* — `docker inspect` for exit code and OOMKilled, logs tail, and a probe from inside the namespace. In this shape — 'Server started' then silence — the classic root cause is the healthcheck *being the restart trigger with a warmup window mis-calibrated* (start_period too short / interval+timeout > actual readiness time), *or* a genuinely wedged loop from the new in-memory cache loading synchronously for the first 30s on the CPU thread. Either way the fix is architectural, not a restart-count tweak: separate liveness (process-heartbeat, cheap) from readiness (real, DB+cache warm), make the health endpoint do a *bounded* self-check returning 503 when truly not ready, and move cache warmup to run asynchronously after `listen` so readiness comes true promptly. In the first 10 minutes I'd remove the failing replica from the LB, stop the autoscale from feeding it pods, and freeze the evidence (thread dump, stats, logs) before changing the probe. And I'd treat 'tuning the probe to paper over the app' as a config change that needs an incident follow-up, because the 25-second window is a signal about warmup design, not about alert thresholds."

## Architect-Level Answer

"This is where the platform meets the application's contract, and the fix is systemic. Containers must expose a *two-signal contract*: liveness = 'keep me', readiness = 'send me traffic', and the healthcheck must not be a shotgun HTTP case that conflates them. As the architect I'd make the runtime consistency platform-enforced: read and liveness probes belong in the platform's golden deployment template, with agreed conventions (paths, timeouts, start-periods) that every service inherits, and I'd add a canary gate: a deploy is *only* promoted if the new image's readiness latency and warm-time stay within the SLO percentiles measured for the previous release. The app's job is to serve readiness as soon as honestly possible — cache warmup is explicitly staged or on-demand, never a synchronous block in the serving thread, because 'warning-free crash' is a correctness property of every service.

Strategically, the restart loop is a symptoms of a *release process that validates builds but not running services* — so I'd invest in environments where the exact container runs under synthetic load with health asserted for a realistic warm cycle, before the promo to prod. And operationally I'd make restart-reason a first-class metric (exit code + OOMKilled + health-probe-failure as labeled signals) feeding both the SLO and the incident triage, so that when an on-call sees '30s restarts' they already have the reason and the warmup comparison in front of them rather than a 3 AM puzzle."

## Follow-Up Questions

1. Distinguish *liveness* from *readiness* precisely. In K8s, liveness probes failing trigger a restart, while readiness failing only removes traffic. Under what *specific* conditions would you deliberately accept liveness failures and *not* restart — and what K8s field makes that possible?
2. Your health endpoint returns HTTP 200 even when the DB connection pool is exhausted (all checked-out). Design a healthcheck that *honestly* reports readiness but does not cause a restart loop for a *transient* DB blip — where exactly do liveness and readiness boundaries split here?
3. `docker run --restart=always` + a crashing healthcheck creates a 30s loop. Explain precisely how `--restart` interacts with the healthcheck result in the Docker daemon: does a failed healthcheck *cause* the restart, or does the restart come from somewhere else? (Hint: think about what actually exits.)
4. You find `start_period: 10s` but the app takes 45s to warm. Diagnose *exactly* when a healthcheck passes and fails with `interval: 30s, timeout: 5s, retries: 3`, and decide how to change *each* of those four fields — or explain why you might choose *not* to change the interval at all.
5. In a crash loop, why might the **first** thing an on-call does be wrong — scaling the deployment up? Explain the restart-storm amplification pattern on K8s (pods × failed LivenessProbe × LB reconnect) and how `maxUnavailable`, `progressDeadlineSeconds`, or a `PodDisruptionBudget` changes the blast radius.