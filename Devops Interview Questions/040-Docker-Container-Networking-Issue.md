# 040. Docker Container Cannot Connect to Another Container

## Scenario

Container A is a Node.js REST API (`node-api`, running on port 3000) and Container B is a PostgreSQL database (`postgres-db`, on port 5432). Both are attached to the same user-defined bridge network named `app-net`.

Yesterday everything worked. Today, `node-api` cannot connect to `postgres-db:5432`. The API logs show:

```
error: connect ECONNREFUSED 172.18.0.3:5432
error: connect ETIMEDOUT 172.18.0.3:5432
```

Some deployments also report:

```
Error: getaddrinfo ENOTFOUND postgres-db
```

The Ops team insists "no network configuration was changed," but the following happened over the last 24 hours:
- A container was recreated (`docker-compose up -d` did a `docker compose down` + `up` after a config tweak; the DB container name stayed the same).
- One engineer "cleaned up docker resources" with `docker system prune -a`.
- The `node-api` container config was edited to add a new environment variable for a feature flag.

The connection config in the API reads:

```
PGHOST=postgres-db
PGPORT=5432
```

The API is failing to start fully because it cannot connect to the database. The load balancer in front of `node-api` is healthy, but every request that touches the DB ends up 500.

## Interviewer Question

"Container A cannot connect to Container B on port 5432. Both are on the same Docker network, and 'nothing was changed' — yet it worked yesterday and fails today, with errors ranging from ECONNREFUSED to ENOTFOUND. Walk me through your troubleshooting methodology: the exact `docker` commands you'd run in which order, why each one matters, and how you'd determine whether the problem is the network, the DNS resolution, the service, the recreated container, or the firewall."

## What I Should Think About

- **There are many distinct failure modes masquerading as one symptom** — "cannot connect" vs "host name not resolved" are different diagnoses:
  - **ENOTFOUND / address not found** → DNS resolution failure → custom network / DNS server / container name + network scoping mismatch.
  - **ECONNREFUSED** → DNS worked, TCP got RST → nothing listening *on that address:port*, wrong port, or listening on 127.0.0.1 only.
  - **ETIMEDOUT** → SYN sent, no SYN-ACK → firewall (host), network isolation, wrong IP/port route, kernel iptables issue.
- **Container-to-container connectivity relies on:**
  1. Both containers on the **same user-defined network** (default bridge ≠ embedded DNS resolution between names).
  2. The `docker network connect` membership / `network_mode`.
  3. The **embedded DNS** (127.0.0.11) resolving container names → fails if containers are on *different* networks or if you used the *default bridge*.
  4. Published ports: inter-container traffic does **not** go through published ports — it goes to the container IP directly. `-p 5432:5432` is irrelevant for container-to-container unless crossing network boundaries.
- **The `docker system prune -a` is the number-one suspect for "database disappeared"** — `prune -a` removes *all unused images* and, critically, if the DB container was stopped/removed it may have been purged along with its *unnamed* volumes (anonymous volumes). Named volumes survive prune; anonymous ones can be cleaned.
- **DNS caching** on the client: Node.js has 9222-style DNS caches (actually legacy `dns.setDefaultResultOrder`) but more likely the *recreated* container has a new IP; Docker's embedded DNS re-resolves per query so a stale IP is rare — unless the client itself caches the resolved IP.
- **Check connectivity at the right layers**: ping (layer 3), nc/curl to port (layer 4), then the app config (layer 7). Always `ping <container-name>` from *inside* A to see DNS + ICMP, then `nc -vz postgres-db 5432`.
- **Common "secret" gotchas**: API listens only on `127.0.0.1` inside the container (should be `0.0.0.0`), Postgres `pg_hba.conf` restricts `host all all <subnet>/16` to a network that changed (new container IP), or the API container joined a *second* default-bridge network that doesn't share DNS.

## Ideal Answer

"I'd approach this as a layered network triangle — the three corners are *networks*, *DNS*, and *the listening service* — and eliminate them one layer at a time with live proof, not assumptions.

**Step 1 — confirm they're actually on the same network.** `docker inspect node-api` and `docker inspect postgres-db`, filter `NetworkSettings.Networks`. The container names must appear under the *same* network ID. A common break: the engineer recreated `node-api` with `--network default-bridge` or left `postgres-db` on a leftover named network. If they're not on the same user-defined network, embedded DNS doesn't even resolve names — and that instantly reproduces ENOTFOUND.

**Step 2 — check whether the DB container is alive and reachable from A's network namespace.** From *inside* container A:
```bash
docker exec node-api ping -c 3 postgres-db
docker exec node-api sh -c "nc -vz postgres-db 5432"
```
If ping works but `nc` fails on 5432, we're at the service layer, not the network. If ping fails to resolve the name, the DNS path is broken.

**Step 3 — check what postgres actually listens on.** `docker exec postgres-db sh -c "ss -ltnp"` or `netstat -tlnp`. A Postgres bound to `127.0.0.1` inside its own container will refuse connections from *other* containers (ECONNREFUSED to the outer IP) even though it looks healthy to the DB owner. In Postgres, `listen_addresses` must be `'*'` or the container IP, not `'localhost'`.

**Step 4 — look for the prune/unplanned change.** If using compose, verify the running config (`docker compose config`) or inspect both networks for members. Check whether the *DB was recreated* with a *new name*, or whether `docker system prune -a` wiped the image so `docker-compose up` silently **recreated the DB with an anonymous volume** → **virgin empty database** — then the API "can't connect" because Postgres exists but auth/db-name vanished (that's a *different crash*, but same symptom). This produces a panic in a new dimension: the answer isn't "network," it's "data loss" — recovering from a backup.

**Step 5 — resolve by construction.** The permanent fix is health-based ordering + stable IPs via a custom network, plus named volumes, plus never `prune -a` in prod without a tag/budget check.

If it's a genuine network-corruption smell (both names resolve, both on the network, nothing listening-only-localhost) then: `docker network inspect app-net` for the IP buckets, force-recreate only the API container, and if that doesn't help, `docker network prune -f` (after checking nothing else uses it) then recreate the network."

## Architecture

```
HAPPY PATH (user-defined bridge network):
┌───────────────────────────────────────────────────────────┐
│   "$ docker network create app-net"                        │
│   app-net (bridge, embedded DNS 127.0.0.11)                │
│   ┌─────────────────────┐      ┌──────────────────────┐   │
│   │ node-api            │      │ postgres-db           │   │
│   │ 172.18.0.2          │      │ 172.18.0.3            │   │
│   │ PGHOST=postgres-db  │─────►│ listen_addresses='*'  │   │
│   │ → 127.0.0.11 DNS    │ TCP  │ pg_hba allows subnet  │   │
│   │ → resolves name     │ 5432 │ │  172.18.0.0/16     │   │
│   └─────────────────────┘      └──────────────────────┘   │
│   name resolution & routing = FREE on a user bridge        │
└───────────────────────────────────────────────────────────┘

BROKEN (one of 3 common breaks):
 A) Different networks → DNS can't resolve name (ENOTFOUND)
 B) Same net, but A.bound-IP ≠ B (ECONNREFUSED), postgres listening only
    on 127.0.0.1 (inside) or mysql skips 172.18.0.x
 C) pruned/DB recreated → new image, anonymous volume wiped →
    DB empty, auth/user gone → connection errors that smell like
    network but are data erasure
```

## Investigation

1. **Enumerate all networks and containers.**
   ```bash
   docker network ls
   docker ps -a --format 'table {{.Names}}\t{{.Networks}}\t{{.Status}}'
   ```
2. **Confirm membership of both containers.**
   ```bash
   docker inspect node-api --format '{{range $k,$v := .NetworkSettings.Networks}}{{$k}} {{end}}'
   docker inspect postgres-db --format '{{range $k,$v := .NetworkSettings.Networks}}{{$k}} {{end}}'
   ```
3. **From inside A, test name resolution vs routing:**
   ```bash
   docker exec node-api ping -c 3 postgres-db
   docker exec node-api getent hosts postgres-db
   docker exec node-api sh -c "nc -vz postgres-db 5432"   # or curl
   ```
4. **From inside B, check listener:**
   ```bash
   docker exec postgres-db sh -c "netstat -tlnp || ss -ltnp"
   docker exec postgres-db psql -U appuser -c "SHOW listen_addresses;"
   ```
   If `listen_addresses` shows `127.0.0.1`, that's the ECONNREFUSED cause.
5. **Verify the DB register / datastore itself** — the 'prune' attack:
   ```bash
   docker inspect postgres-db --format '{{.Mounts}}'     # named vs anonymous volume
   docker volume ls
   docker volume inspect <name>                          # survives prune
   ```
   If `Mounts` has an anonymous hash-named volume → data gone after prune.
6. **Check pg_hba for subnet mismatch** — postgres inside container:
   ```bash
   docker exec postgres-db sh -c "cat /var/lib/postgresql/data/pg_hba.conf | grep -A 10 'IPv4'"
   ```
   If it only grants `172.18.0.0/16` (or old subnet) and the network changed to `172.19.0.0/16`, connections fail at auth rather than network.
7. **Check the API-side connection caching/retries:**
   ```bash
   docker inspect node-api --format '{{range .Config.Env}}{{println .}}{{end}}' | grep -i PG
   docker logs node-api --since 10m | tail -40
   ```
   Look at whether it resolves to stale IP; a proxy/connection-pool with a cached IP is a classic after `docker system prune` (containers recreated, IPs changed).
8. **Rule out host iptables / risk surface:**
   - If A and B are on *different* bridge networks, DOCKER chain port translation may block cross-network — `docker network connect app-net postgres-db` if needed.
   - `docker network inspect app-net` shows IPAM config and currently attached endpoints.

## Commands

```bash
# Level 0 — inventory
docker network ls
docker ps -a --format 'table {{.Names}}\t{{.Networks}}'

# Level 1 — same network membership?
docker inspect node-api    --format '{{json .NetworkSettings.Networks}}'
docker inspect postgres-db --format '{{json .NetworkSettings.Networks}}'

# Level 2 — DNS written inside A's container
docker exec node-api getent hosts postgres-db        # name→IP via embedded DNS
docker exec node-api cat /etc/resolv.conf

# Level 3 — TCP reachability (layer 4)
docker exec node-api sh -c "nc -vz postgres-db 5432; echo exit=$?"

# Level 4 — actual listener state inside B
docker exec postgres-db sh -c "ss -ltnp | grep 5432"
docker exec postgres-db psql -U postgres -c "SHOW listen_addresses;"

# Level 5 — did the DB get recreated/pruned (data loss risk)?
docker inspect postgres-db --format '{{json .Mounts}}'
docker volume ls | grep -E 'pg|postgres'
docker ps -a --filter name=postgres-db --format '{{.Image}} {{.CreatedAt}}'

# Level 6 — fix: ensure both on the same user network
docker network create app-net
docker network connect app-net postgres-db
docker network connect app-net node-api

# Level 7 — live reproduction test after fix
docker exec node-api sh -c "nc -vz postgres-db 5432"

# Debug helper: with the DB on the same net but inside a different subnet,
# confirm with the IP directly:
docker exec node-api sh -c "nc -vz 172.18.0.3 5432"   # vs name above
```

## Root Cause

- **Containers on different networks / default-bridge vs user network** — Docker's embedded DNS (127.0.0.11) only resolves container names on *user-defined* bridge networks; on the legacy default bridge, name resolution doesn't work. Recreating `node-api` with `--network host` or the default bridge while the DB is on `app-net` yields ENOTFOUND.
- **`docker system prune -a` / recreation wiped data** — if the DB container used an *anonymous volume* (attached only by the container), prune removes the unreferenced volume and the recreated DB boots into a clean, empty data directory. The API connects (TCP-wise) but fails auth (that user/db is gone) → error messages that *look* like connectivity. This is the dangerous assumption to catch: "can't connect" ≠ "data intact."
- **Postgres `listen_addresses='localhost'` / bound to 127.0.0.1** — inside its own container, localhost-only listening means traffic from *other* containers on the Docker bridge is refused at the socket level (ECONNREFUSED) even though `docker ps` shows the DB "running."
- **pg_hba.conf subnet drift** — honestly `172.18.0.0/16` allowed before; after the recreate, the network got a new subnet (172.19.0.0/16): auth fails → the API says "connection failed" though SSH/TCP works. pg_hba must match the container subnet or `%` + strong password.
- **Client-side DNS/stale-IP caching** — some clients (connection pools, `pgBouncer`, HAProxy static config) cache the DB's old container IP; after a prune/recreate the IP changed, so TCP times out (ETIMEDOUT) even though DNS works for fresh queries.
- **The host firewall / security group** — on some setups inter-container traffic crosses host iptables. Docker normally auto-manages this; a Docker daemon restart or SELinux/AppArmor intervention can break the FORWARD chain. Rare, but test with `iptables -L FORWARD` and check daemon logs if all in-namespace checks look good.

## Immediate Mitigation

1. **Diagnose-first check the data before touching anything**: if `docker system prune -a` happened, immediately check whether the DB volume exists (`docker volume ls`). If the DB *is* gone: **fail-safe priority is restoring the database** (backup with last snapshot/recovery) since data loss is worse than downtime. If a different container's local DB was affected, use the latest backup/replica to sync it.
2. **Put both on the same network** as a surgical fix:
   ```bash
   docker network connect app-net node-api    # adds node-api to DB's network
   ```
   Verify with `docker exec node-api ping postgres-db`.
3. **Point the API at the right thing** if the name is the problem: if names can't be fixed instantly, temporarily set `PGHOST` to the container-real-IP of `postgres-db` (`docker inspect postgres-db --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'`) — acceptable as a stop-gap in a single-host pattern, *never* as a permanent fix (IPs drift).
4. **Restart the API in a controlled way**: `docker compose up -d --force-recreate node-api` (after the network fix) so its DNS cache and connect-pool reset cleanly.
5. **If DB is missing (prune case)**: restore from the most recent backup first, then ensure the container attaches the *named* volume before starting the API. Never restart the API without confirming the DB data directory is the intended one.

## Permanent Fix

1. **Use only user-defined networks + explicit Compose config**, never rely on default-bridge for app stacks:
   ```yaml
   services:
     postgres-db:
       networks: [app-net]
     node-api:
       networks: [app-net]
   networks:
     app-net:
       driver: bridge
   ```
2. **Named volumes, always.**
   ```yaml
   services:
     postgres-db:
       volumes:
         - pgdata:/var/lib/postgresql/data
   volumes:
     pgdata:
   ```
   Anonymous volumes die in `docker system prune -a`; named volumes and explicit `-v` survive, and prune can be made safe with `-a --filter "label=..."` + `--volume` risk checks, and never running interactive `docker system prune -a` from an on-call.
3. **Postgres config contract in code**: `command: -c 'listen_addresses=*'` in the compose service; pg_hba documented to use CIDR of the app subnet; scanner in CI: alert on `listen_addresses=localhost`.
4. **Health-gated startup ordering** (see Q044) so API never *starts querying* before DB is fully ready:
   ```yaml
   node-api:
     depends_on:
       postgres-db:
         condition: service_healthy
   ```
5. **Make the app resilient to transient DB network events**: connection retry with backoff, pool sizing, and prior-host list — so a 1-minute DB restarter doesn't crash the API; implement `/healthz` that only marks ready when the DB pool is warm.
6. **Network recon via compose config as a review item**: `docker compose config` diff in CI (or `docker network inspect app-net`) gates changes, so nobody silently moves a container off the shared network.
7. **Hint for the Ops-team blunder class**: add a CI lint/forbidden command — block `docker system prune -a` in prod (script audit/notification), or use a "prune with filter" policy.

## Monitoring

- **Network-layer:**
  - `docker network inspect app-net` → endpoint list, IPs, subnet — a drift check (new IPs on every recreate).
  - `ethtool`/`ip` in-node: bridge stats & RX/DROP counters on `docker0`/`br-<net>` (`ip -s link show br-…`).
- **Container-level:**
  - Connection pool metrics from the app (`pg_pool_available`, pool waiters) — alerts on `pool.peek_avg` and connection failures.
  - DNS resolution failure rate (log-inside dns) → `pglookup`-like metric; alert on ENOTFOUND episodes.
- **App + DB:**
  - `pg_stat_activity` connections & dead connections; `pg_stat_database` on the app DB name — dropped/crashed connections spike after inne bundle restart.
  - Failure alert: errors matching `ECONNREFUSED|ETIMEDOUT|ENOTFOUND` from the API → page on out-of-band.
- **Operational:**
  - Container recreate/network-change events (`docker events --filter type=network`) to SIEM so 'someone restarted/cleaned' is visible in the timeline, not a rumor "nothing changed."
  - Volume existence/backup freshness probe (0 if unattached) so a prune-event can never silently nuke the data backend again.

## Security

- **Inter-container network is a trust zone you must segment**: not every service belongs on every network. This is where security-balance enters — don't over-consolidate "everything connected to everything." Keep decoupled: DB on `db-net` (only reachable by the API), API on `web-net` (reachable by LB); document CIDR rules in the network plan.
- **Never expose Postgres to the host without necessity**: publishing `5432` to the host / any interface is a giant attack surface. Keep DB on internal bridge, don't publish unless absolutely needed (and then `127.0.0.1:5432:5432` + firewall).
- **pg_hba** is the firewall of last resort — use strongest auth (`scram-sha-256`), restrict to app subnet, and never `host all all 0.0.0.0/0 md5`.
- **The DNS answers are poison to clients**: on user-defined bridges, Docker's DNS returns *all* aliases; a malicious container with the same alias can hijack a name. Pin by digest + use `network_alias` sparingly; consider a service-mesh (Sidecar DNS/`telepresence`) for untrusted workloads.
- **Audit log & forensics**: `docker events` + daemon journal ship evidence of container recreation, network attach and prune — critical for a security incident ("who pulled this container down?").

## Production Considerations

- **HA**: DB on a named volume + dedicated network survives container recreation, enabling smooth rolling restarts; the API should tolerate DB restarts via pool + backoff rather than crash-looping.
- **Scalability**: multiple API replicas all resolve `postgres-db` → same embedded-DNS answer → load toward a single Postgres on the plane; if the DB needs connection-pooling, gateway it (pgBouncer) inside the same net for n× connections.
- **Reliability**: name resolution is a single point of failure on Docker embedded DNS (everything breaks if `--network` is different); making network+compose the durable "source of truth" removes the whole class.
- **Cost**: an unnecessary public port on the DB can cause egress/data-exfiltration costs and is a NAT/traffic risk; keep it internal for both security and egress cost.
- **Compliance**: audit trails of container-network membership and volumes are needed for incident reconstruction; store `docker inspect` at promote-time as "release-side evidence" for SOC2/PCI review.
- **Operational**: every On-call playbook for "container can't connect" should *start* with `docker inspect` on both containers (networks + mounts) before any kind of config dental-surgery; include `docker system prune` as forbidden in prod, promoted to a runbook-blocking policy.

## Senior-Level Answer

"I've seen 'can't connect between containers' silently mean five different things: wrong network membership (ENOTFOUND), service bound only to localhost (ECONNREFUSED), a `docker system prune -a` that recreated the DB with an anonymous volume so the *data* disappeared and auth broke — the most dangerous one to misjudge — plus subnet drift in pg_hba and stale client IP caches. My process is strictly layered: first confirm both containers list the same user network in `docker inspect`; then from inside A, test `ping postgres-db` (DNS), `nc -vz postgres-db 5432` (routing); then from inside B, check `ss -ltnp` for the actual listener and `listen_addresses`. The single most decisive thing: if a prune happened, check `docker volume ls` and the container's `Mounts` for an anonymous volume before anything else — because restoring data outranks restoring connectivity. The permanent fix is architectural: one user-defined bridge per stack declared in Compose, named volumes for everything persistent (so prune can't erase state), Postgres `listen_addresses=*` inside the stack, `depends_on: condition: service_healthy`, app-side retry with backoff, and limiting `docker system prune -a` to environments where data-loss is acceptable. Diagnose in that order and you convert a 3 AM 'nothing changed' incident into a reproducible root cause within minutes."

## Architect-Level Answer

"This symptom is the signature of a handcrafted wiring contract — the team wrote connection strings against container *names and IPs* and trusted implicit Docker defaults, and that dependency graph is fragile. Architecturally I'd move every inter-container dependency to a *declared networking contract* expressed in the platform's source of truth: per-application user networks (never the default bridge), named volumes so that data has an identity independent of container lifetime, DNS name references only (never IPs), and health-gated start ordering so a client never begins traffic to a service that is still booting. Compose/K8s Services are the enforcement point, and the recreate-volatile 'works yesterday' class of failure disappears.

Strategically, the biggest signal here is organizational: 'nothing was changed' is never true, and the platform should make *all* runtime mutations observable — `docker events`, network-membership drift checks, volume-freshness monitors, and a change-log so incidents open with a diff, not a whodunnit. And I'd harden the data plane: as the team grows, the Postgres sits behind a connection pooler (pgBouncer) on its own network with strict pg_hba; the API pool tolerates DB restarts; backups are tested for the 'anonymous volume lost' scenario specifically, because that failure is silent and far more expensive than a mere connectivity outage. The network fix is the patch; the durable platform capital is in name-based routes, volume identity, observable change history, and blast-radius segmentation — those are what let a busy on-call answer the question 'what do I break by fixing it?' in seconds."

## Follow-Up Questions

1. Both containers *claim* to be on `app-net` after `docker inspect`, but `ping postgres-db` from inside the API still fails to resolve. List at least three things that would produce this state (think: network alias, iptables FORWARD chain, Docker DNS config in compose, stale daemon state) and how you'd isolate each.
2. `docker system prune -a` was run in prod. The Postgres *container* is rebuilt, connects, but every query fails with "role does not exist." Reconstruct the exact sequence of events that led here and describe the *data-loss* recovery decision tree — backup source, restore target, and how you test the restored DB before reopening traffic.
3. Container-to-container traffic does **not** pass through published ports on the same network. Prove it: describe packet flow from API→DB over Docker's user bridge — where is the DNAT/MASQ applied, and when would `-p 5432:5432` actually change the client's perceived connectivity (e.g., across *host* networks)?
4. The API uses a connection-pool with `pgBouncer` as a sidecar next to the API, while the DB is on a *different* network. Draw the correct network topology and explain which services each pooler can reach; then explain why the pooler *must* live on the same overlay/bridge as its clients, and what happens to DNS resolution for `postgres-db` from the pooler.
5. `docker compose up -d` recreated `node-api` but left `postgres-db` untouched. Explain Docker Compose's *name resolution / container-renaming* behavior and its risk: what exactly happens to DNS aliases and IPs when only one side of a pair is recreated, and how would you use `docker compose up --no-recreate` or network-scoped `service_healthy` gates to avoid the momentary mismatch?