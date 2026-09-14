# 044. Docker Compose Services Not Starting in Order

## Scenario

The development and staging teams are all reporting the same issue: every time they run `docker-compose up -d`, the Node.js API starts before PostgreSQL is ready. The API logs:

```
Error: connect ECONNREFUSED 172.20.0.2:5432
    at TCPConnectWrap.afterConnect [as oncomplete]
Attempting to reconnect in 5 seconds...
Error: connect ECONNREFUSED 172.20.0.2:5432
```

The API crashes or retries for 30 seconds until Postgres finishes initializing, then reconnects — but the healthcheck fails during those 30 seconds, the container is marked unhealthy, and the load balancer marks the service as down. On some machines the race is worse: Postgres is slower to start (smaller host, no SSD), and the API retries exhaust.

The team's current `docker-compose.yml`:

```yaml
version: "3.8"
services:
  api:
    build: .
    ports: ["3000:3000"]
    depends_on:
      - postgres
  postgres:
    image: postgres:14
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
```

The team says "`depends_on` is there, what else do you want?" — and they're confused because `depends_on` *does* wait... but only for the container to be *created*, not for Postgres to be *ready*.

The other problem: every time the API fails to connect at startup, it writes a stack trace to the log, triggering the PagerDuty alert. The team has been ignoring the alert for two weeks, which is itself a new problem (alert fatigue).

## Interviewer Question

"Docker Compose `depends_on` waits for the *container to start*, not for the *service to be ready*. Your team's app crashes at startup because Postgres isn't ready yet. Walk me through exactly how `depends_on` works (all three modes), what `condition: service_healthy` does, how to implement a proper healthcheck for Postgres, and how you'd design the whole startup sequence to be both *reliable* and *resilient* — including what the app should do when its dependency *is* temporarily unavailable."

## What I Should Think About

- **`depends_on` has three behaviors in Compose v2+ (Compose Spec):**
  1. **`depends_on: [postgres]`** (bare list) → the API container is *created after* the postgres container is *created*. Postgres is created almost instantly (Docker creates the container object) but the service isn't initialized yet. This is the current (broken) setup.
  2. **`depends_on: postgres: condition: service_started`** (explicit) → same as bare list: waits for container to be started (process begins). Equivalent to (1).
  3. **`depends_on: postgres: condition: service_healthy`** → waits until the healthcheck on Postgres returns `healthy`. This is the correct pattern: the API container is only started after Postgres's healthcheck passes.
  - There's also a **`restart: true`** field that re-evaluates if the dependency restarts (Compose v2.20+).

- **Why `depends_on` alone doesn't work:** Docker Compose orchestrates container *creation*, not service *readiness*. `depends_on: postgres` ensures the postgres container is created (and its process started) before the API container is created — but Postgres takes 2–15 seconds to actually initialize (create data dir, load config, accept connections). During that window, the API connects → ECONNREFUSED.

- **The Postgres healthcheck:** `pg_isready -U postgres` is the canonical probe. It returns 0 if Postgres is accepting connections. The correct configuration is:
  ```yaml
  postgres:
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
      start_period: 10s
  ```
  `start_period` is critical: Postgres needs ~5–10 seconds to initialize a fresh data dir; during `start_period`, failed healthchecks don't count toward `retries`. After `start_period` ends, 5 consecutive failures = unhealthy.

- **Resilience inside the app:** even with `service_healthy`, transient failures happen (network blip, Postgres restart, rolling update). The app *must* have:
  - Connection retry with exponential backoff.
  - A startup retry loop that blocks readiness until the DB is first connected.
  - Graceful degradation if the DB is temporarily unreachable (503, not 500).
  - Pool management: connection pool timeout, retry on acquire failure.

- **The alert fatigue problem:** the team has been ignoring PagerDuty for two weeks because the startup stack traces are "expected" — they're not bugs, they're a design flaw. The fix is two-pronged: (1) make the startup reliable so the stack traces don't happen; (2) if they still happen (Postgres *is* legitimately down), suppress the alert or route it to a non-paging channel.

- **Compose v2 vs v3:** the `condition: service_healthy` feature was removed in Compose v3 (YAML version `3.x`) and restored in the Compose Spec (Docker Compose v2 CLI, `docker compose` without the hyphen). In legacy `docker-compose` v1, the syntax was `depends_on: postgres: condition: service_healthy`. The team's `version: "3.8"` may be hiding that `condition` is ignored in v3 — this is a common gotcha.

- **Order of other dependencies:** if the API depends on both Postgres *and* Redis, both need healthchecks; the API waits for *both* to be healthy. Document the full dependency graph.

## Ideal Answer

"The core misunderstanding is conflating `depends_on: [postgres]` with 'Postgres is ready.' It isn't. `depends_on` with a bare list only guarantees that the API *container is created after the Postgres container is created* — not that Postgres has finished initializing, loaded extensions, created roles, or is accepting TCP connections. You can verify this by watching `docker-compose up` and seeing the API start its Node.js process at the same time the Postgres logs show `database system is starting up`.

The fix is `condition: service_healthy` paired with a real healthcheck on Postgres. Here's the correct setup:

```yaml
services:
  api:
    build: .
    ports: ["3000:3000"]
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      DATABASE_URL: postgres://postgres:secret@postgres:5432/app
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:3000/health || exit 1"]
      interval: 10s
      timeout: 3s
      retries: 5
      start_period: 20s

  postgres:
    image: postgres:14
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: app
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d app"]
      interval: 5s
      timeout: 5s
      retries: 5
      start_period: 10s

volumes:
  db_data:
```

The Postgres healthcheck uses `pg_isready`, which returns exit code 0 only when Postgres is accepting connections — not just when the container process is running. `start_period: 10s` gives Postgres a grace window to initialize before failures count.

The API container won't start until Postgres's healthcheck returns `healthy` — so the ECONNREFUSED at startup disappears.

**But I also make the app resilient to transient failures.** Even with perfect `depends_on`, a network blip or Postgres restart happens. So the API has:
- Connection retry with exponential backoff (not 'retry every 5s forever,' but 'try 1s, 2s, 4s, 8s, up to 30s max, then fail with a clear 503).
- A startup gate: the first thing the app does on boot is attempt a DB connection; it doesn't register with the LB until that succeeds.
- Graceful degradation: if the DB drops mid-request, return 503 (not crash), keep the health endpoint alive, let the LB drain and retry on another replica.

And the **alert fatigue problem**: if the team has been ignoring PagerDuty for two weeks because of startup errors, that means either (a) the startup errors are expected behavior (the alert is wrong — fix the alert threshold or route it to a non-paging channel), or (b) the startup errors are a real bug being normalized (fix the root cause, which is the missing `service_healthy`). The correct fix is both: fix the root cause, and then if a 503 *does* happen, decide whether it's worth paging a human (a transient 503 lasting <30s is not page-worthy; a 503 lasting 10 minutes is)."

## Architecture

```
BEFORE (depends_on: bare list → container created, not ready):

Compose startup timeline:
t=0   postgres: container CREATED, process STARTS → initializing...
t=0   api: container CREATED (depends_on satisfied), process STARTS
t=0.1 api: connects to postgres:5432 → ECONNREFUSED (not yet listening)
t=1   api: retry #1 → ECONNREFUSED
t=3   api: retry #2 → ECONNREFUSED
t=8   postgres: log "database system is ready to accept connections"
t=8.5 api: retry #3 → connected ✓
      (8.5 seconds of startup failures, stack traces, pager alerts)


AFTER (condition: service_healthy):

Compose startup timeline:
t=0   postgres: container CREATED, process STARTS → initializing...
      postgres healthcheck starts firing at t=10 (start_period)
t=10  postgres healthcheck: pg_isready → not ready yet (start_period, not counted)
t=15  postgres healthcheck: pg_isready → ready ✓ → status: healthy
t=15  api: depends_on condition satisfied → container CREATED, process STARTS
t=15.1 api: connects to postgres:5432 → ✓ connected immediately
      (0 seconds of startup failures)


Resilient API startup (defense-in-depth):
┌────────────────────────────────────────────────────────────┐
│ API container process                                       │
│ 1. Attempt DB connection (with 3s timeout, 3 retries)      │
│ 2. If connection OK → register with LB → start serving      │
│ 3. If connection FAIL → sleep 5s → retry (max 30s total)   │
│ 4. If still FAIL after 30s → log FATAL + exit (fast fail)  │
│    → orchestrator restarts with backoff                     │
│ 5. Mid-life: DB blip → pool retry, 503 on exhausted pool   │
│    → no crash, health stays green, LB retries elsewhere     │
└────────────────────────────────────────────────────────────┘
```

## Investigation

1. **Confirm the actual `depends_on` mode in use.** Open `docker-compose.yml` and check: bare `depends_on: [postgres]` or `depends_on: postgres: condition: service_healthy`? Then check the Compose file version — if `version: "3.x"` (v3 YAML spec), the `condition` field is **ignored by design**. You must use the Compose Spec format (no `version:` or `version: "3.8"` with Compose v2 CLI, or switch to `version: "2.4"` for legacy `docker-compose` v1).
   ```bash
   docker compose config 2>/dev/null | grep -A5 "depends_on"
   ```

2. **Check Postgres's actual readiness timeline.** Watch the logs and timing:
   ```bash
   docker-compose up 2>&1 | tee /tmp/startup.log
   ```
   Note the timestamp when the API starts vs when Postgres logs `ready to accept connections`. The gap between those timestamps = the vulnerable window.

3. **Confirm the Postgres healthcheck is actually running and configured correctly.**
   ```bash
   docker inspect <postgres_container> --format '{{json .Config.Healthcheck}}'
   docker inspect <postgres_container> --format '{{.State.Health.Status}}'
   ```
   If `Healthcheck` is null/empty, no healthcheck was applied — the `condition: service_healthy` has nothing to wait for.

4. **Test the healthcheck from inside Postgres manually.**
   ```bash
   docker exec <postgres_container> pg_isready -U postgres -d app
   ```
   Exit code 0 = ready; exit code 1 = not ready. This is what the orchestrator watches.

5. **Check the API's connection retry behavior.** Open the API source code and look for the DB connection logic: does it crash on first failure, retry N times, or have a proper retry loop? This determines whether `service_healthy` is sufficient or if you also need app-side resilience.

6. **Evaluate the alert noise source.** If the API logs the ECONNREFUSED as an unhandled exception → uncaughtException → process exits → restart → repeat, then every restart generates a stack trace → PagerDuty alert. This is the alert fatigue source. Fix: the DB connection logic should catch the error, log a warning (not a stack trace), retry, and only log an error/exit after exhausting retries.

7. **Confirm no other services have the same race condition.** `grep -n "depends_on" docker-compose.yml` — check every service for bare `depends_on` without `condition: service_healthy`. Redis, RabbitMQ, Elasticsearch are equally vulnerable.

## Commands

```bash
# 1. What does Compose think depends_on does for each service?
docker compose config | grep -A 8 "depends_on"

# 2. Is the healthcheck actually applied to postgres?
docker inspect <postgres_container> --format '{{json .Config.Healthcheck}}'
docker inspect <postgres_container> --format 'Status={{.State.Health.Status}}'

# 3. Watch the health status transition in real time
docker compose ps
# NAME          STATUS
# postgres      running (healthy)   ← this is what depends_on waits for
# api           running

# 4. Manually probe from inside postgres (is pg_isready working?)
docker exec <postgres_container> sh -c "pg_isready -U postgres -d app; echo exit=$?"

# 5. Check the startup gap in the logs
docker compose logs --timestamps --tail 50 postgres | grep "ready to accept"
docker compose logs --timestamps --tail 50 api | head -5

# 6. The correct docker-compose.yml (Compose Spec, no version: needed)
cat docker-compose.yml

# 7. For legacy docker-compose v1 with v2 format (condition: service_healthy)
# ensure file starts with:
#   version: "2.4"
# and then:
#   depends_on:
#     postgres:
#       condition: service_healthy

# 8. Test the full startup from scratch (full recreate)
docker compose down
docker compose up -d
docker compose ps          # watch: api should not start until postgres: healthy
docker compose logs api    # should NOT show ECONNREFUSED
```

## Root Cause

- **`depends_on` with bare list (the #1 issue):** the team uses `depends_on: [postgres]` which waits for *container creation*, not *service readiness*. Postgres's internal initialization (data dir, `initdb`, config loading) takes seconds; during that window the API connects and gets ECONNREFUSED.
- **Compose file `version: "3.x"` silently ignores `condition: service_healthy`:** Docker Compose v3 (the YAML format `3.x`) intentionally removed the `condition` field to stay compatible with Docker Swarm, which doesn't support it. Even if the team had `condition: service_healthy`, the `version: "3.8"` header makes Compose silently ignore it. The fix is either: use the Compose Spec (no `version:` field), or use `version: "2.4"` with legacy `docker-compose` v1.
- **No healthcheck defined on Postgres (or healthcheck exists but returns the wrong thing):** `condition: service_healthy` has nothing to wait for if the healthcheck is `null`, or if `pg_isready` is run with wrong credentials/database name, or if the `CMD-SHELL` form silently fails (common if `pg_isready` isn't installed in the image).
- **App has no startup retry logic:** even with perfect `depends_on`, the API crashes on first connection failure instead of retrying. A transient failure at any time (Postgres restart, rolling update, network blip) produces the same stack trace, same alert, same alert fatigue.
- **Alert fatigue from expected failures:** the team ignores PagerDuty alerts because the stack traces happen *every deploy* and are "expected." This is a normalization-of-deviance problem: a real bug (missing retry + wrong depends_on) was reclassified as "normal" instead of fixed, and the alerting was not adjusted. Now a *real* failure (Postgres data corruption, disk full) would be ignored alongside the noise.
- **`start_period` not configured (or misconfigured):** without `start_period`, the first healthcheck runs immediately after the container is created; Postgres almost always fails that first check → if `retries: 3` and `interval: 5s`, the container is marked unhealthy at t=15s before Postgres is even done initializing.

## Immediate Mitigation

1. **Fix the Compose file now** — swap `depends_on` to `condition: service_healthy` and add a real healthcheck. The fastest path is using the Compose Spec format:
   ```yaml
   services:
     api:
       depends_on:
         postgres:
           condition: service_healthy
     postgres:
       healthcheck:
         test: ["CMD-SHELL", "pg_isready -U postgres -d app"]
         interval: 5s
         timeout: 5s
         retries: 5
         start_period: 10s
   ```
   Remove the `version:` field (Compose Spec) or set `version: "2.4"` (legacy `docker-compose` v1).

2. **Suppress the pager alert for the startup window** while the fix is deployed. If PagerDuty is wired to every log error, add a filter or tag so that `ECONNREFUSED` during the first 30s of a container lifecycle is downgraded to a non-page (e.g., Slack warning), not a PagerDuty incident.

3. **Restart the stack to validate the fix.**
   ```bash
   docker compose down && docker compose up -d
   docker compose ps    # confirm: api waits until postgres: healthy
   ```

4. **If Postgres is slow to initialize on some hosts** (small host, spinning disk, no SSD): increase `start_period` and `retries` to give Postgres more breathing room. Monitor `pg_isready` timing across hosts to find the right values.

5. **Add connection retry to the app as a parallel fix** — don't rely solely on Compose. Even with `service_healthy`, a transient DB blip after startup would crash an app without retries. Add an exponential backoff retry with max 30s timeout on the DB connection at startup.

## Permanent Fix

1. **Standardize the Compose dependency pattern for the team** — document and template:
   ```yaml
   services:
     api:
       depends_on:
         postgres:
           condition: service_healthy
         redis:
           condition: service_healthy
     postgres:
       healthcheck:
         test: ["CMD-SHELL", "pg_isready -U postgres -d app"]
         interval: 5s
         timeout: 5s
         retries: 5
         start_period: 10s
     redis:
       healthcheck:
         test: ["CMD", "redis-cli", "ping"]
         interval: 5s
         timeout: 3s
         retries: 5
         start_period: 5s
   ```
   Every service that depends on another gets a `condition: service_healthy` gate.

2. **Require a healthcheck on every stateful service in CI.** A lint rule (e.g., `docker compose config` + `yq` or a custom check) that fails the pipeline if a service with a volume or external dependency has no `healthcheck` defined. This prevents future services from being added without health-aware startup.

3. **App-level resilience as a non-negotiable standard:**
   - Connection retry with exponential backoff (1s, 2s, 4s, 8s, capped at 30s).
   - Startup gate: don't register with LB until the first successful DB ping.
   - Graceful degradation: DB unavailability → 503 with retry-after header, not a crash.
   - Pool management: connection acquire timeout, pool refresh on error.

4. **Fix the alert fatigue at its source:** once the app retries gracefully, the ECONNREFUSED stack traces disappear. For the transitional period, reclassify the alert: the startup error is a warning-level event (Slack), not a PagerDuty incident, until the fix is deployed.

5. **Test the startup order as part of CI/CD.** A CI job that runs `docker compose up -d` and waits for all services to reach `healthy` status within N seconds, then runs a smoke test against the API. If the API fails to become healthy within the timeout, the pipeline fails. This catches regressions before they reach staging/prod.

6. **Document the startup dependency graph** in the repo README or a service catalog: which services depend on which, what the healthchecks check, what `start_period` values are set, and what the expected startup timeline is. New team members and on-call engineers can read this without guessing.

7. **Version discipline:** pin the Compose file format to the Compose Spec (no `version:` field for `docker compose` v2+) so the `condition` field is always honored. If the team must support legacy `docker-compose` v1, use `version: "2.4"` — never `version: "3.x"` when `condition: service_healthy` is needed.

## Monitoring

- **Startup timing:** track the time from `docker compose up -d` to all services `healthy` as a metric; alert if it exceeds the expected window (e.g., 30s) — this catches hosts with slow disk or Postgres initialization regressions.
- **Health status per service:** `docker compose ps` health status → expose via `cadvisor` or a custom exporter; a service stuck in `unhealthy` for > 5 minutes should page (it means the healthcheck is failing *after* start_period, not during startup).
- **Connection retry events:** track the number of DB connection retries per API startup; alert if it's > 0 (the fix should eliminate retries entirely; any retries indicate a residual race condition).
- **Alert on actual failures, not startup noise:** once the race is fixed, remove the ECONNREFUSED alert from PagerDuty; replace with a `5xx_rate` threshold (e.g., > 5% of requests return 5xx for 2 minutes → page). This eliminates the alert fatigue while catching real production failures.
- **Dependency health degradation:** a service that was healthy and then becomes unhealthy (e.g., Postgres runs out of disk) should produce a distinct alert from "startup not ready" — the former is a production incident; the latter is a transient startup event.
- **Dashboard:** a "startup health" panel showing `container_health_status{status="healthy"}` vs `status="starting"` vs `status="unhealthy"` over the last 7/30 days — regression in startup time or healthcheck failure rate becomes visible before it causes an outage.

## Security

- **Healthcheck endpoints can expose internal state** — `pg_isready` is safe (returns a generic ready/not-ready), but custom healthchecks that expose connection strings, database names, or component versions to a probe endpoint are an information-leak risk. Keep healthchecks simple and internal.
- **Credentials in healthcheck commands** — if the healthcheck is `CMD-SHELL "psql -U postgres -p secret"`, the password is in `docker inspect` (visible to anyone with Docker socket access). Use `PGPASSWORD` env var inside the container instead of inline password.
- **Healthcheck access control** — a health endpoint exposed on `0.0.0.0:3000/health` is a reconnaissance surface. If possible, health probes should run from *inside* the container (Docker's healthcheck), not from external port scans. This limits visibility to container-local or orchestrator-local probes.
- **`pg_isready` trust** — it checks if Postgres is accepting connections, not if the database is *functional* (data corruption, full disk). A more robust healthcheck does `SELECT 1` as the database user, confirming both connection *and* query capability.
- **Dependency availability as a security boundary** — if the API crashes on DB unavailability, an attacker can DoS the API by taking down Postgres (or even just slowing it). Resilient retry + graceful degradation means the API stays available for non-DB routes even when the DB is unreachable.
- **Compose secrets** — `POSTGRES_PASSWORD: secret` in plain text in the file is a security anti-pattern. Use `env_file`, Docker secrets, or an external secrets manager. The healthcheck doesn't need the password, but the service config shouldn't have it in plaintext.

## Production Considerations

- **HA:** `depends_on` is a single-host concept. In a multi-host cluster (K8s, Swarm), the dependency model changes — you need `initContainers` (K8s) or service-mesh readiness gates. Understand that `docker compose up -d` on a single host doesn't translate to distributed orchestration; plan for the transition.
- **Scalability:** `depends_on` doesn't scale with `docker compose up --scale api=5`. Scaling happens after the dependency is resolved, but if Postgres is the bottleneck, 5 concurrent API startups may exhaust its connections. Add a connection pool limit or stagger startup.
- **Reliability:** the `service_healthy` condition is the *minimum* — for production-grade startup, the app should have a "service-init" phase that validates all dependencies before accepting traffic (startup probe, readiness gate, or init container).
- **Cost:** every extra healthcheck poll (interval: 5s) uses a tiny amount of CPU; in a fleet with 50+ services, the cumulative cost of healthchecks is non-trivial but overwhelmingly worth it for the reliability. Tune `interval` to balance responsiveness vs cost.
- **Compliance:** startup behavior is a SLA commitment. If the API takes 30 seconds to become ready after a deploy, that's part of the deployment window. Document it so the SLO is honest.
- **Operational:** the `docker compose config` command is your friend — always run it before deploy to verify that the effective YAML (merged overrides, resolved values) matches intent. A `depends_on` that was silently ignored by the wrong `version:` header is exactly the kind of thing `config` reveals.
- **Migration to Kubernetes:** the Compose `depends_on` pattern maps to K8s `initContainers` + `readinessProbe`. If the team is migrating to K8s, design the healthchecks *now* in a way that translates: the same `pg_isready` command works in both `healthcheck.test` (Compose) and `readinessProbe.exec.command` (K8s).

## Senior-Level Answer

"The root cause is a Compose file that uses `depends_on: [postgres]` — which waits for the *container to be created*, not for Postgres to be *ready to accept connections* — combined with a `version: "3.8"` format that silently ignores `condition: service_healthy` even if someone added it. The API starts while Postgres is still initializing, connects to a port that isn't listening yet, crashes, retries, and generates stack traces that became PagerDuty noise the team learned to ignore. The fix is three layers: (1) switch to Compose Spec (drop the `version:` field or use `version: "2.4"`) and use `condition: service_healthy` with a real Postgres healthcheck (`pg_isready -U postgres` with `start_period: 10s`); (2) make the app resilient to transient DB unavailability with connection retry and graceful degradation — because `service_healthy` doesn't prevent a mid-life Postgres restart; (3) fix the alert fatigue by removing startup ECONNREFUSED from PagerDuty and replacing it with a `5xx_rate` threshold that pages on real production failures. The `depends_on` misunderstanding is common precisely because Docker Compose gives you a *feeling* of orchestration while really only handling container lifecycle — not service readiness — and the `version: 3.x` silent ignoring of `condition` is the trap that catches teams who think they fixed it."

## Architect-Level Answer

"This is a systems-design failure at the dependency-contract level: the team assumed the runtime orchestrator (Compose) understood service readiness, while Compose only understands container lifecycle — and the `version: 3.x` format made the mistake invisible by silently ignoring `condition: service_healthy`. The architectural fix is to formalize the contract between every service: each service that has an external dependency declares a *healthcheck* that proves readiness, and each consuming service declares a `condition: service_healthy` gate in its dependency. This makes the startup graph explicit and machine-enforced rather than implied and undocumented.

At scale, this pattern becomes the platform's responsibility: every service that joins the platform gets a golden healthcheck template (Postgres → `pg_isready`, Redis → `redis-cli ping`, etc.), the platform validates that the healthcheck exists before the service is deployed, and the orchestrator enforces the dependency graph. This turns a per-service manual fix into a platform guarantee — new services inherit the right pattern automatically.

The strategic evolution: when the team moves to K8s, the same health-check contract maps cleanly to `readinessProbe` + `initContainers`, and the dependency graph moves to a service mesh (linkerd/istio) that can do deep dependency probing. But even at the Compose level, making startup-order a *documented, validated, and tested* property of the stack — not a 'we hope it works' assumption — is what separates a team that has startup incidents from one that never sees them. The alert fatigue is the other signal: it's the team's way of saying 'we've normalized a bug as expected behavior,' and that normalization is itself the deeper risk — it makes a *real* failure invisible behind the noise of a known one."

## Follow-Up Questions

1. Compose `version: "3.x"` silently ignores `condition: service_healthy`. Explain *why* this design decision was made (Docker Swarm compatibility) and describe the exact two ways to get `condition` working again — using Compose Spec (no version field) and using `version: "2.4"` with legacy `docker-compose` v1. What happens if someone uses `version: "3.8"` with `docker compose` v2 CLI?
2. `pg_isready` confirms Postgres is accepting connections but doesn't confirm the *database exists*. Design a more thorough healthcheck that validates connection + database existence + query execution. What's the trade-off between healthcheck thoroughness and startup delay?
3. Your team has 12 microservices in a single Compose file. The dependency graph is complex: A→B, A→C, B→D, C→D, D→E. Draw the dependency graph, determine the correct startup order, and explain how Compose resolves the `depends_on` graph when there are circular or diamond-shaped dependencies.
4. The API starts before Postgres is ready, crashes, and the orchestrator restarts it. After 3 restarts in 30 seconds, Docker marks the container as `restarting` and stops trying. Explain Docker's restart backoff behavior (what's the delay between restarts?), and how `depends_on: condition: service_healthy` changes this behavior compared to `restart: always` without healthchecks.
5. You're migrating from `docker compose` to Kubernetes. Map the following Compose concepts to their K8s equivalents: `depends_on: condition: service_healthy` → ?, `healthcheck: test` → ?, `start_period` → ?, `restart: always` → ?, and `depends_on` (bare list) → ?. What K8s primitive solves the "Postgres not ready" problem *without* an explicit healthcheck?