# 042. Docker Volume Data Loss After Container Recreation

## Scenario

During a routine deployment, the platform team runs `docker-compose up -d --force-recreate` on the production stack. The Postgres-backed app and a file-storage module come back up, but the entire persistent dataset is gone — the SQLite/DB got reset **and** all uploaded user files vanished.

The stack looked like this:

```yaml
services:
  app:
    image: app:1.0.0
    volumes:
      - ./data/app.db:/app/data/app.db
      - ./uploads:/app/uploads
  db:
    image: postgres:14
    environment:
      POSTGRES_PASSWORD: something
    volumes:
      - /var/lib/postgresql/data          # ← anonymous volume!
```

What actually happened (reconstructed after the fact):

1. A previous engineer had set the Postgres data volume up as an **anonymous volume** (`/var/lib/postgresql/data` — no name, no host bind). The container held the only reference to that volume.
2. Someone ran `docker system prune` / `docker-compose down -v` (or the deploy script had `down -v`), which removed the anonymous volume.
3. `docker-compose up -d --force-recreate` created a *brand-new* anonymous volume → **fresh empty Postgres data dir**.
4. The app's SQLite DB (bind-mounted `./data/app.db`) survived on the host — but the **files** in `./uploads` were bind-mounted from a dev checkout that was overwritten/removed on the next `git clean`/deploy, so uploads are gone too.
5. Buckets/backups exist only for the managed DB, not for the on-host file and DB volumes — so restore is a scramble.

Now the on-call must figure out two things: **where the data went** and **how to bring the application back**. User account data and documents are in SQLite and the filesystem; Postgres had the transactional core which is now empty.

## Interviewer Question

"A container was recreated during a deployment and all persistent data was lost — the SQLite database file and uploaded files are gone. Explain the *different persistence models* in Docker (named volume, anonymous volume, host bind mount), which ones survive `docker-compose down -v` and `docker system prune` and which don't, why the data disappeared in this scenario, and then show me the correct Compose file with a data-loss-proof persistence design (volumes, backups, recovery)."

## What I Should Think About

- **Docker's three persistence models and their lifetimes:**
  - **Named volume** (`pgdata:/var/lib/postgresql/data`) — survives container recreation and `docker-compose down` (without `-v`); survives `docker system prune` (does **not** remove named volumes by default). Cleanup only via `docker volume rm <name>` over it.
  - **Anonymous volume** (`/var/lib/postgresql/data` template-form or `-v container:/path`) — created with a hash name, tied to the container; removed by `docker-compose down -v` and by `docker system prune` **with `-a`/`--volumes`** when unreferenced. **This was the trap.**
  - **Host bind mount** (`./data:/app/data`) — continues to exist (it's a host dir), but *risks*: a `git clean -fdx`, a fresh checkout on a CI agent, `rm -rf data/`, or a directory wiped by a deploy script destroys it; also chmod/users differ per host.
- **The lifecycle killers:**
  - `docker system prune -a -f` → removes all unused images + **stopped containers** + dangling networks + *unused anonymous volumes*.
  - `docker-compose down -v` → explicitly removes volumes created by the file: **all anonymous + named volumes in the compose file**. This is the exact pair that "lost" the Postgres data.
  - `docker-compose down` (no `-v`) → keeps volumes.
  - `docker-compose up --force-recreate` → recreates the container but **reuses** the same volume list *if* the volume definitions still exist.
- **Bind mounts are only as durable as the host path:** on prod they're a *host dependency* (must be on the host filesystem, the host upgrade, backup). The directory + perms are a "hidden contract" — the platform must back them up, like a volume.
- **The correct production pattern:**
  1. Data → **named volumes** declared in `volumes:` block, referenced by name.
  2. **Never rely on anonymous volumes or implicit `-v`** for anything you care about.
  3. **Backup as a first-class job** (a cron sidecar: `pg_dump`/`pg_dumpall` to object storage, or a volume snapshot; SQLite backup with `sqlite3 .backup`, files bucket backup).
  4. Stateless **recreate**; stateful **migrate** — the recreate loop must be the documented path for stateless services only.
  5. Add a "guard" so a fresh-data-dir Postgres fails loudly rather than happily booting empty (e.g., fail if DB not initialized, or restore job runs before app).
- **Recovery, immediately:** stop the app, do NOT let an empty Postgres accept writes, restore Postgres from backup (there *may* be a prior snapshot), recover SQLite + uploads from the *host* path `data/` and `uploads/` if they exist (check `.gitignore`! someone may have left them on a host); if truly gone, restore from external snapshots; after restore, switch the compose to named volumes + automatic backup job.
- **The deeper lesson** to communicate: "recreate == data risk" and "prune == data risk"; the platform should log both operations and make `-v` disallowed in prod (a policy guard in the deploy tooling).

## Ideal Answer

"Docker gives you three persistence primitives, and the *first mistake* in this stack is that the Postgres data relied on an anonymous volume — a hash-named directory attached only to the container. The container's `volumes: - /var/lib/postgresql/data` declaration *looks* like persistence, but the volume has no name and no external reference, so the moment anything removes the container's sole reference (or `docker-compose down -v` / `docker system prune -a` cleans unused volumes), the data directory is gone as if it never existed.

Second, the app data was a **bind mount** (`./data/app.db` + `./uploads`), which is durable *only while the host path survives* — and this deploy's `git clean`/fresh-checkout step wiped or replaced the working tree directory. So one half of the data vanished through the volume model (Postgres, anonymous), the other through host-path volatility (SQLite + uploads).

**The correct design:**

```yaml
services:
  app:
    image: app:1.0.0
    volumes:
      - app_data:/app/data
      - uploads:/app/uploads
    depends_on:
      db:
        condition: service_healthy
  db:
    image: postgres:14
    environment:
      POSTGRES_PASSWORD: ...
      PGDATA: /var/lib/postgresql/data/pgdata
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  app_data:
  uploads:
  db_data:
```

Key rules I enforce:
1. **Named volumes only** in the declaration — the volume's identity is independent of the container; recreate reuses it; `down` (no `-v`) keeps it; prune leaves it.
2. **Never put `down -v` in a prod deploy script.** If you need it (destroy-env), it's a separate destructive command with a visual confirmation and a backup gate.
3. **Bind mounts only for config/secret injection** (`-v /etc/foo:/etc/foo:ro`) or in dev; state lives in named volumes or the object store.
4. **Backup is a platform job, not a human memory.** A schedule must snapshot DB + files to object storage (see Commands), and the restore procedure must be tested — the worst thing worse than losing data is losing the *ability to restore* the backup you thought you had.
5. **Make 'empty fresh data' loudly bad.** Postgres mounts a new data dir happily; the production guard should detect 'no initial data' and either fail the boot or run the restore path — silent empty-boot means the same symptom repeats as a quiet data wipe instead of a pager.

For the recovery in this specific incident: I'd stop the app immediately (before any new writes stamp the empty DB), attempt the host-path recovery for SQLite (`./data/app.db` may still exist in `.git`-ignored dirs on the prod host or a snapshot), restore Postgres from the latest backup if one exists, and re-point the compose to named volumes for the future. And I'd change the guardrail that "someone ran the deploy the way the docs said" — because our docs now forbid both anonymous volumes and `down -v` in production."

## Architecture

```
DATA-LOSS CHAIN (this incident):
container lifecycle
┌──────────────┐ ref(only) ┌────────────────┐ down -v / prune
│ container    │──────────►│ /var/lib/pg/…  │──────────────► ✗ gone
│ (db)         │  holds it │ anonymous vol  │
└──────────────┘           └────────────────┘
bind mounts: ./data/app.db, ./uploads ── host dir ── git clean
                                                        ────► ✗ wiped

WHAT SURVIVES WHICH COMMAND:
+---------------------+-----------+-------------+----------------+-----------+
| primitive           | down      | down -v     | system prune -a| prune -a  |
|                     |           |             | (stopped)      | +volumes  |
+---------------------+-----------+-------------+----------------+-----------+
| named volume        | ✓ stays   | ✗ removed   | ✓ stays        | ✓ unless  |
|                     |           |             |                | unrefer'd |
| anonymous volume    | ✓ if ref  | ✗ removed   | ✓ if attached  | ✗ likely  |
| (still attached)    |           |             |                | removed   |
| host bind mount     | ✓ stays   | ✓ stays     | ✓ stays        | ✓ stays   |
+---------------------+-----------+-------------+----------------+-----------+

CORRECT PERSISTENCE TOPOLOGY:
┌───────────────────────────┐
│ prod host                 │
│  ┌─────────────────────┐  │  ┌──────────────────────┐
│  │ named volume        │  │  │ recovery/backup job  │
│  │ app_data, uploads,  │  │  │ (sidecar/cron)       │
│  │ db_data             │  │  │ pg_dump → S3/Blob    │
│  └──────────▲──────────┘  │  │ tar `uploads` → obj  │
│             │ mount       │  └──────────▲───────────┘
│  ┌──────────┴──────────┐  │             │
│  │ containers (stateless)│             │ restore drill
│  │ recreate freely     │  │             │
│  └─────────────────────┘  │             │
└───────────────────────────┘     ┌──────────────┐
                                  │ object store │
                                  └──────────────┘
Recreate keeps state: name → same volume dir; down -v forbidden in prod.
```

## Investigation

1. **Confirm what was destroyed first.** Look at the deploy script / CI step for `down -v` or `prune`:
   ```bash
   grep -rn "down -v\|system prune" deploy/ .gitlab-ci.yml Makefile
   ```
   If it's there, the "how" is answered — data-loss-by-policy.
2. **Inventory volumes before/after:**
   ```bash
   docker volume ls
   docker volume inspect <name>   # driver, mountpoint, labels
   docker inspect <container> --format '{{json .Mounts}}'
   ```
   A *recently created* anonymous volume name + `Labels["com.docker.compose.volume"]` tells you when it was spun.
3. **Check for a live unnamed volume ("the data may not be gone yet"):**
   - If prune removed it: on the host filesystem, the actual volume dirs live under `/var/lib/docker/volumes/` (or the manager's data-root). Look for orphaned dirs still on disk — prune *removes the volume record*, sometimes leaving the data dir:
   ```bash
   ls -la /var/lib/docker/volumes/ | grep X   # orphan dirs
   ```
   (On Linux hosts rarely recoverable; on Docker Desktop WSL you may still retrieve from the WSL disk.)
4. **Recover the bind mounts** — the SQLite + uploads: they exist as *host directories* — maybe still on the deploy host behind a `git clean -fdx`? Check:
   ```bash
   find / -name "app.db" -o -name "uploads" 2>/dev/null   # careful, targeted find
   git stash list; git reflog   # if a checkout overwrote
   ls -la .gitignore            # did the repo even ignore ./data and ./uploads?
   ```
5. **Postgres restore pass:** if a named volume or backup exists `pg_dumpall` restore: `psql -f backup.sql`. If none, check the database server's crash-recovery logs; a *fresh init* produces a different state than a *restore* (the original was likely `/var/lib/postgresql/data` from the image's first boot manual volume).
6. **Forensics of the actual sequence:** timestamps on volume dirs, container `CreatedAt`, and the deploy log tell the exact chain (prune → recreate → empty PGDATA). This is the "where did it go" narrative, not just "it's gone."
7. **Assess blast radius on the object-store** (if any backup exists) — check S3/Blob retention ("is there a full + PITR?"). This decides whether recovery is minutes or hours.
8. **Write the incident report skeleton immediately**: start/end times, commands run, volume/model at fault, backup status, restore path taken — while memory (and the host filesystem) is still fresh.

## Commands

```bash
# 1. Any losing command in your deploy path?
grep -rniE "down -v|system prune|--volumes" Makefile deploy/ .github/ 2>/dev/null

# 2. Volume inventory + owners
docker volume ls
docker inspect <container> --format '{{json .Mounts}}'

# 3. Are there orphaned data dirs in the daemon root? (best-effort recovery)
sudo ls -la /var/lib/docker/volumes/ | grep -iE "pg|sqlite|upload"
# Docker Desktop: check WSL disk
wsl ls -la /var/lib/docker/volumes/

# 4. Hunt the bind-mount paths (SQLite + uploads)
find /app /data /srv -maxdepth 4 -name "app.db" -o -name "uploads" 2>/dev/null
git -C .loma reflog | head -20        # if checkout overwrote a tracked dir
git -C /path/to/prod checkout -- data 2>/dev/null || echo "checkout not needed"

# 5. Restore Postgres from backup (if a dump exists)
psql -h db -U app -d app < /backups/app_$(date +%F).sql

# 6. The fix — named volumes + backup sidecar (compose)
cat docker-compose.prod.yml
# (see Ideal Answer) named volumes: app_data, uploads, db_data

# 7. Backup sidecar (cron) — dumps db + tars uploads to object storage
docker run --rm -it --network host \
  -v pgdata:/var/lib/postgresql/data:ro \
  -v upl:/app/uploads:ro \
  -e AWS_ENDPOINT=... \
  alpine:3.18 sh -c '
    pg_dumpall ... | gzip > /tmp/db.gz && \
    tar -czf /tmp/up.tar.gz /app/uploads && \
    mc cp /tmp/db.gz /bk/bucket/db/$(date +%F).gz'

# 8. Restore drill (tested annually)
docker stop app db
docker run --rm -v db_data:/vol alpine:3.18 sh -c \
  'mc config ... ; mc cp bk/bucket/db/$(date +%F).gz /vol/latest.gz && \
   psql ... '
docker start db app     # verify queries = 0 row mismatch
```

## Root Cause

- **Anonymous volume as the DB's persistence (the fatal design error)** — `- /var/lib/postgresql/data` in the container's `volumes:` (or `-v /container/path` on the CLI) creates an *unnamed* volume owned only by this container. When the container is removed and the compile-plugin/`docker compose down -v` or `docker system prune -a` runs, nothing references the volume → it's garbage-collected → data destroyed.
- **`docker-compose down -v` / `system prune -a` in the production path** — a destructive, environment-reset command promoted into prod as routine deploy. Its correct home is CI/dev-local destroying a *throwaway* environment; in prod it requires a consent gate / dry-run "backup check."
- **Bind-mounted state tuned for dev checkout that's volatile in prod** — `./data/app.db` and `./uploads` assume the host path survives. Prod deployments using `git clean -fdx` or fresh-checkout CI agents wipe those directories — in effect state on a *recycled* surface.
- **No backup as a twin requirement** — neither the Postgres (which had no external snapshot) nor the SQLite/uploads had a scheduled backup to object storage. Any persistence design without a backup is a design with a future outage — usually the *silent* kind: the app boots fine, "looks healthy," while the data library below is empty.
- **Silent-fail boot of an empty datastore** — a fresh `PGDATA` directory inits to a new empty cluster without complaint; "container healthy" ≠ "data present". Without a "data-presence" guard, a recreated DB can pass all health and readiness probes while serving nothing.

## Immediate Mitigation

1. **Freeze.** Immediately stop the app and any DB-init / migration jobs (`docker compose stop app`) so no writes land on the (possibly empty) new datastore before recovery. Identify all replicas / secondaries before acting — do not let an autosync fan out the empty dataset.
2. **Rescue the big three in order — data, restore-path, then uptime:**
   - Postgres: restore from the latest `pg_dump`/snapshot/`PITR` if a backup exists. If not, hunt for the orphaned volume dir on the host/WSL (`/var/lib/docker/volumes/<hash>`), and pause briefly before restoring; consider disk-forensics if critical (stop the engine... no writes).
   - SQLite: check the host paths `./data`/`./uploads` and any snapshot (LVM/cloud disk snapshot) — attach that snapshot, copy `app.db` + files out.
   - **Only after the data is back and verified** (row-count parity vs the alert expectation) bring the app up and re-run migrations/backfill if needed.
3. **Prove integrity before announcing**: run a query-count sanity (`SELECT count(*)` on tables, `ls uploads | wc -l`) vs expected business numbers; if the restore is off by orders of magnitude, it's not a restore yet.
4. **Binary decision — do NOT silently accept empty restore:** if no backup exists and all recovery paths fail, you have declared a data-loss incident: page the owner, set up the mis-cut writeup, and prevent the empty DB from being marked "healthy" — the app must not accept traffic on it.
5. **Change the tooling so the same deploy can't do this again tonight**: remove `-v`/prune from the prod path, and switch the compose to named volumes as the immediate "next-deploy" fix.

## Permanent Fix

1. **Named volumes are the only production persistence model** — declare them explicitly in a `volumes:` top-level in compose; reference by name:
   ```yaml
   volumes:
     app_data: {}
     uploads: {}
     db_data: {}
   ```
   And use them in every service. Never anonymous. Never "I'll just add `-v /path` to a container."
2. **Guard the destructive commands at the tooling and culture level:**
   - `down -v`, `system prune -a --volumes` → blocked/flagged in prod deploy scripts (a pre-commit hook or pipeline gate greps the deploy manifest).
   - Add an explicit "pre-destroy data check" — a job asserting that all named volumes present + backup freshness age < SLA before any destroy operation.
3. **Backup as a first-class production object (SAAS-style):**
   - Postgres: nightly `pg_dumpall` + continuous WAL archiving (or managed snapshot) → object store, retention ≥ 30 days, tested restore quarterly.
   - Files: `uploads/` volume → `rclone`/`mc` sync to object storage nightly; keep a versioned bucket (objects change).
   - SQLite (if kept): `sqlite3 .backup` nightly, or replace it with the managed DB — the point is *documented, tested restores*, not volumes alone.
4. **Add the "data-presence" guard to stateful containers**: a startup check that refuses to boot a *new* empty data dir unless a restore flag is explicitly present (`/var/lib/.../INITIALIZED` marker, PG's `PGDATA` init behind a wrapper). Fresh-boot-healthy-empty is the exact failure you must make loud.
5. **Make recreate safe-by-construction**: stateless services are free to `--force-recreate`; stateful services must have a narrow, reviewed migration path (named volume + pre-run DB health + migration job that fails rather than recreates).
6. **Annual restore drill** — a full dry run restoring DB + uploads to a staging compose from the object store; document the measured RTO/RPO. If it isn't drivable, it's not a backup, it's a hope.
7. **Write the runbook**: "Data missing after container recreation" → freeze → eh... restore → verify → re-point to named volumes → report. Deliver it to on-call in the incident postmortem.

## Monitoring

- **Volume/identity signals:**
  - Volume-event stream (`docker volume ls` diff, or the events API) — alert when *any* volume under `db_data`/`uploads` disappears while the service is up.
  - Per-volume "last data write" freshness: if `uploads/` stays empty for N days while traffic says otherwise → likely silent loss.
- **Data-presence probes:**
  - For Postgres: a job every 5 min running `SELECT 1` + row-count on a sentinel table → alert if query succeeds but table count = 0 (fresh-empty signature).
  - For files: `uploads/` file count diff vs last count; alert on unexpected drop.
- **Backup health:**
  - Backup job success/failure (`backup_last_success{age}`) — alert if older than RPO-target.
  - Restore drill smoke test tied to the alert — the "validated" badge.
- **Change/ops:**
  - Deploy steps producing `down -v`/`prune` → a compliance/audit event deflected to the queue — the "guard" should itself be observable.
- **Business-level:**
  - "App data correctness" synthetic=left-to-right: user-created file count (via API) ≈ `ls uploads | wc` — the crash of the two is the earliest silent-loss signal.

## Security

- **Anonymous-volume data loss is a forensic black hole** — after GC the data may be unforensically unrecoverable; for regulated data (PII), losing the volume requires a *data-breach *assessment, not just an outage note. Document the model before the incident so a data-loss event is handled as reported, not discovered later.
- **Protect secrets stored in volumes!** `.env` / keystores / certs living in a volume are not "private" to the mount — with bind mounts they're host-world-readable and with anonymous volumes they're daemon-root-owned. Always `:ro`, permissions per user, never commit.
- **The restore path must not reintroduce old secrets...** restoring an old DB betrays secrets/certs rotated since; the restore procedure must run a rotation task afterwards (rotate keys used by the recovered DB, invalidate old sessions/tokens that may still be valid).
- **Volume encryption-at-rest** for anything sensitive (Docker does not encrypt volume contents by default) — use encrypted filesystems (LUKS) or cloud-encrypted EBS keying, and never rely on "the volume is a dir" for compliance.
- **Multi-tenant blast radius**: the same mis-modeled anonymous volume in a shared fleet could leak *co-tenant* data when pruning; enforce "named + labeled volumes only" as a security policy, not a convenience.
- **Audit trail**: volume create/attach/detach/remove events, and the deploy commands, must appear in the SIEM for a future "who killed the data" review.

## Production Considerations

- **HA**: a named volume does NOT provide HA — it's single-host storage inside one Docker daemon. For real HA move Postgres to a managed service/hosted disk with cross-AZ replication; the volume is the durability layer, not the HA layer. The platform should communicate that distinction clearly.
- **Scalability**: named volumes are host-local; containers must be scheduled onto the node that owns the volume (or use a clustered storage driver like Rook/Gluster — with all their complexity). Understanding this is what separates *named-volume* designs from real *distributed-datastore* designs for scale.
- **Reliability**: durability != availability. The freeze-then-restore path above is the honest "RTO = restore time"; publish the real RTO/RPO from the drill, not the optimistic one.
- **Cost**: object-store backup for high-churn uploads can get pricey; store compressed, versioned, with lifecycle rules (30d hot / 90d cold / archive), or the "upload bucket" becomes a hidden bill. Set the retention policy on day one.
- **Compliance**: every data class stored (PII, transaction, audit) needs defined retention & deletion; anonymous-volume GC is *not* a compliant deletion method (it's an incident). Named volumes + explicit policy make audits clean.
- **Operational**: the #1 operational win is the "no destructive command in prod" guard and the annual restore drill — both are cheap, and both convert this recurring firefight into a boring, documented procedure.

## Senior-Level Answer

"This outage is a persistence-model bug, not an 'oops.' The Postgres data was an *anonymous volume* — `- /var/lib/postgresql/data` in the service's volumes — which exists only as long as a container references it, so `docker-compose down -v` or `docker system prune -a` garbage-collects it with the container. The app's data was a *bind mount* from `./data` + `./uploads`, which survives `down`/`prune` but not a `git clean` or a fresh checkout on a recycled host dir. Both halves of the persistence were designed wrong: one had no external identity (anonymous ⇒ disposable), the other had no host durability contract (bind ⇒ depends on a path nobody else guarantees).

The fix is a standard I'd enforce platform-wide: **state lives in named volumes only**, declared in `volumes:`, referenced by name, so recreation always reuses the same storage; `down -v` and `prune -a` are banned from prod deploy paths; bind mounts are used only for read-only config injection; every stateful service pairs with a nightly object-store backup job and a *tested* restore, because the true failure here isn't the volume model — it's that nobody had a working restore path. And the last guard is the subtle one: a recreated DB boots a fresh empty cluster and passes every health probe — so I add a 'data-presence' check that fails the boot when the data directory isn't the *initialized* one, turning a silent data wipe into a loud pager first."

## Architect-Level Answer

"Architecturally I separate three concerns the incident conflated: **durability** (is state on non-volatile storage?), **identity** (can I reattach the same state after recreate?), and **recoverability** (can I rebuild it after absolute loss?). The production design must answer each independently: durability → named volumes or managed disk; identity → volume names and labels survive container churn; recoverability → tested object-store backups with documented RPO/RTO. And the platform encodes that answer where the mistakes happen — in the deploy tooling: destructive commands (`down -v`, `prune -a`) are blocked in the prod pipeline, and any stateful service starts a "pre-flight" check (data present + backup fresh) before release.

The strategic shift this incident funds is moving state *out* of anonymous ephemeral hosting entirely: Postgres should be a managed service or the platform's own HA datastore with replication; `uploads` should be an object store (S3-compatible) from day one; the SQLite file inside a bind mount is a *smell* that belongs in a dev-only stack. Container scheduling, scale-out, and node failure then work naturally because the *volumes* the orchestrator manages are the stateless-plus-object-store pattern, not node-local disk that can't follow the workload. That's the real HA answer: not 'mount the volume twice,' but 'make the critical state live where the platform already has durability, identity, and recovery.' The on-call lesson, though, is universal and cheap: every persistence design ships with a free test — 'if I run `down -v` and `prune -a` right now, what do I lose, and how fast can I get it back?'"

## Follow-Up Questions

1. Compare the exact lifetime of anonymous vs named vs bind-mounted storage against each of: `docker-compose down`, `down -v`, `docker system prune`, `prune -a --volumes`, and a `docker volume rm <name>`. Which *single* command-family is the standard 'accidental prod reset,' and how do you design a deploy script that can never produce it?
2. A volume's `mountpoint` lives at `/var/lib/docker/volumes/<hash>/_data`. After `docker system prune --volumes`, the volume *record* is gone but the data directory might still exist on disk. What does the practical recovery look like (copying `_data` out, re-attaching by hand), under what conditions is it safe, and why is it *not* a substitute for backups?
3. In Compose, `- ./data:/app/data` (bind) vs `- app_data:/app/data` (named). List the *operational differences* beyond survival: user/permission semantics (host vs container UIDs), NFS/point-in-time snapshot behavior, and how each behaves when the compose file moves to a second host or a CI agent.
4. Your new deployment "fixed" the Postgres volume (named, survives everything) but you *still* lost user files because a migration job wrote wrong content into `uploads/`. Walk through the failure modes that named volumes do NOT protect you from (logic errors, ransomware, deleted-by-app, mismerge), and design the defense (versioned bucket, snapshot schedule, cross-region copy) for each.
5. "Let's just put everything in an S3 bucket / managed cloud DB and stop worrying about Docker volumes" — as an architect, evaluate that statement honestly: what do you *gain* (durability, scaling, backups), and what *new* failure modes or costs appear (latency to local writes, egress costs, lock-in)? When is a named volume still the *right* choice for a workload class?