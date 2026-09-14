# 038. Docker Image Size Optimization

## Scenario

Your organization's flagship product is a Java Spring Boot API with a React frontend. Deployed as a single Docker image — a "monolith in a box" containing both the built frontend static assets and the JAR. The image is currently **2.5 GB**.

Symptoms and consequences:

- CI/CD pipeline takes 30+ minutes because pushing and pulling the image from the registry takes 10–15 minutes per environment (dev, staging, prod).
- Storage costs are rising: image registry blob storage, plus each node in the cluster downloads the image independently.
- Cold starts of pods take over a minute because the node must untar a giant image layer set.
- Developer iteration is slow — every local rebuild of the frontend triggers a full image rebuild because layers are invalidated.

The build currently looks like this, and nobody knows why the image is so big:

```dockerfile
FROM maven:3.8-jdk-11 AS build
COPY . /app
WORKDIR /app
RUN mvn clean package
FROM openjdk:11-jdk
COPY --from=build /app/target/app.jar /app.jar
COPY --from=build /app/frontend/dist /static
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

Actually — even simpler, in some places the team builds the app on a dev machine (with full JDK and Maven and node_modules) and then `docker build` on top **using a `COPY . .` that includes `target/`, `node_modules/`, and `.git`**. Files like `target/classes/` and `node_modules/` have tens of thousands of unneeded files, massively inflating layer size.

Interview context: the interviewer asks you to own the image and bring it down to a reasonable size and, conceptually, propose the *right* multi-stage architecture for a Java + React app.

## Interviewer Question

"Your Docker image for a Java Spring Boot backend plus a React frontend is 2.5 GB. Pushing and pulling images are slowing your CI/CD pipeline down to painful levels, and registry storage costs are climbing. Walk me through exactly what you inspect to understand why the image is so large, and then show me the concrete Dockerfile changes you'd make to cut the image (ideally under 300MB) while keeping the builds reproducible. What are the trade-offs?"

## What I Should Think About

- **Why is the image big?** The answer always starts with *layer analysis* — you cannot guess. Use `docker history`, `docker system df`, `dockle`, `trivy`, and modern tools like `docker build --progress` with BuildKit or `dive` to inspect layer sizes and unfreed space.
- **The three main size culprits for Java + React:**
  1. **Wrong base image**: `openjdk:11-jdk` / `maven:3.8-jdk-11` include the whole JDK plus build tools. For runtime only, use the JRE (`eclipse-temurin:11-jre`) — or better, a distroless JRE.
  2. **Build artifacts leaking into runtime**: a naive `COPY . .` copies `target/`, `node_modules/`, `.git`, `~/.m2` caches, possibly the entire live repo — often 10–50x the real app size.
  3. **Single-stage build**: combining build tools (Maven, Node) with the runtime image makes every layer weigh as much as the build environment.
- **Multi-stage builds** solve both (2) and (3) at once: build in a fat stage, run in a thin stage.
- **The React frontend**: build it in a Node stage, copy only the `dist/` output. Options: include static files in the Spring Boot JAR (`spring-boot-maven-plugin` + `src/main/resources/static`) or serve React from a separate nginx container. Choose to match the deployment model.
- **JVM niche consideration**: Java modules — with `jlink` you can create a minimal runtime image that is only JRE-modules you actually use (from ~285MB to ~40MB). Spring Boot supports `--enable-native-image` (GraalVM) but that's a bigger architectural change; mention it but don't over-sell.
- **Layer caching**: order `COPY` operations least-frequently-changing first (dependencies before source) so rebuild-invalidation doesn't force the frontend rebuild on every commit.
- **Sizing targets**: base JRE ~300MB (temurin 11 / 21), thin JAR ~40–80MB, React `dist/` a few MB → indicative 180–300MB total; with `jlink`/distroless → 100–150MB.

## Ideal Answer

"I'd start by *measuring*, not guessing. Three commands to find the bloat: `docker history <image> --no-trunc` shows each layer's size and the build step that created it; `docker system df` shows how much disk is in dangling images; and something like `dive run <image>` or `dive <image>` interactively shows which layers only *add* data and which cause earlier data to be *shadowed* but still kept (that's the classic 'why is my image 1.5GB when files add to 200MB' trap).

What I usually find in a 2.5GB Java/React image is a combination of: (1) a `COPY . .` that drags `target/`, `node_modules/`, `.git` into the image; (2) a full JDK in the runtime stage instead of a JRE; (3) leftover Maven/Node cache directories and `.m2` repositories; and (4) the natural cost of using a `maven` or `openjdk` base instead of a jre/distroless base.

Then I fix it with a proper multi-stage build:

```dockerfile
FROM node:20-alpine AS fe-build
WORKDIR /fe
COPY frontend/package*.json ./
RUN npm ci
COPY frontend/ ./
RUN npm run build

FROM maven:3.9-eclipse-temurin-21 AS be-build
WORKDIR /app
COPY --from=fe-build /fe/dist /app/src/main/resources/static
COPY pom.xml .
RUN mvn -q -DskipTests dependency:go-offline
COPY src ./src
RUN mvn -q -DskipTests package

FROM eclipse-temurin:21-jre
RUN useradd -r appuser
COPY --from=be-build /app/target/app-*.jar /app/app.jar
USER appuser
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

Key rules I enforce: never `COPY . .` at the app root without a `.dockerignore` excluding `.git`, `target`, `node_modules`, `**/tmp`; build frontend in a Node stage and copy just `dist`; build the backend in a Maven stage and copy just the JAR; run on the *JRE*, not the JDK. That typically takes a 2.5GB image to ~250MB with a standard JRE, and below 150MB if I swap to `jlink` or a distroless base.

Two trade-offs I call out: distroless/slim runtimes make shelling into the container harder (no `bash`, no package manager) — debugging changes; and skipping the .dockerignore is the #1 reason layer sizes balloon, so I make `.dockerignore` a review gate. Also: if the team serves the React app via Spring Boot's static resources, the single-image approach is fine and simplest operationally; if the frontend needs independent scaling/CDN behavior, split it into a separate nginx image. Either way the image size and CI speed come straight from the build structure."

## Architecture

```
BEFORE (naive single stage, 2.5 GB):
┌────────────────────────────────────────────────────┐
│ maven:3.8-jdk-11 (JDK + maven + git)               │
│  ├─ COPY . .  → target/, node_modules/, .git,      │
│  │              ~/.m2, .idea, *.log   (1.5GB!)     │
│  ├─ node artifacts, two full toolchains            │
│  └─ openjdk:11-jdk at runtime → needs full JDK     │
│  Total ≈ 2.5 GB                                    │
└────────────────────────────────────────────────────┘

AFTER (multi-stage, ~250 MB):
┌────────────┐   ┌──────────────────────┐   ┌──────────────────────────┐
│ fe-build   │   │ be-build             │   │ runtime                  │
│ node:20    │   │ maven:3.9-temurin-21 │   │ eclipse-temurin:21-jre   │
│ npm ci     │──►│ COPY dist → static   │──►│ COPY only app.jar        │
│ npm run    │   │ mvn package          │   │ USER appuser             │
│ build      │   │ (deps cached)        │   │ ENTRYPOINT java -jar     │
│ → dist/    │   │ → target/app.jar     │   │ ≈ 250 MB                 │
└────────────┘   └──────────────────────┘   └──────────────────────────┘

Optional jlink shrink (~130MB): runtime stage starts
  FROM eclipse-temurin:21-jre, then
  jlink --add-modules java.base,java.sql,...
  --strip-debug --no-man-pages --no-header-files --compress=2
  → small custom runtime, no full JRE.
```

## Investigation

1. **Quantify each layer.** `docker history <image> --no-trunc` → note every `COPY`/`RUN` that adds >100MB. The output column `SIZE` is the current size of the layer contribution; this alone usually identifies the guilty `COPY . .`.
2. **Check for shadowed files.** Because Docker layers are stacked, deleted files are not actually removed from lower layers — the image ships them anyway. Use `dive run <image>` (or `dive <image>`) and press `Tab` to switch between "layer" and "image" views; look at *image tree* for `target/`, `node_modules/`, `.git`, `*.dll`/`*.so`, dumps, logs.
3. **Verify what's copied.** Compare `tar -tf` of a temporary container filesystem vs the actual needed runtime:
   ```bash
   docker create --name tmp-inspect <image>
   docker export tmp-inspect | tar -tf - | sort | head
   ```
   Check if `target/` or `node_modules/` or `.git` directories exist inside the container filesystem.
4. **Check base image fatness baseline.** Build a trivial `FROM <base> CMD ["sleep","999"]` image and measure its size; this gives you the unavoidable base cost and lets you write budget: base (jre) + jar + dist.
5. **Inspect effective .dockerignore.** Check if the repo has one. If missing: `COPY . .` grabs *everything* in build context, including the checked-out `.git` (often 300MB+ alone).
6. **Check the JAR itself.** `ls -lh target/*.jar` — a Spring Boot *fat jar* is typically 40–80MB. If larger, check it's not accidentally bundling node_modules or a second JRE (`unzip -l app.jar | grep -E "BOOT-INF|node_modules"`).
7. **Measure push/pull latency attribution.** `docker pull <image> | tail -1` shows compressed size; registry storage charges on compressed size while nodes unpack to the uncompressed size. Record both to set the optimization target.
8. **If applicable, use `docker build --dockerfile` + full `--progress` to** confirm the pipeline is using BuildKit and that layer caching is actually being used (`CACHED` markers), because long CI runs may also be a caching policy issue, not only size.

## Commands

```bash
# 1. Layer-by-layer size analysis (fast, no extra tools)
docker history <image> --no-trunc | sort -k2 -h | head -30

# 2. Disk usage of docker + dangling images
docker system df

# 3. Interactive deep-dive into layer content (essential tool)
dive <image>          # or: dive run <image>

# 4. List actual files inside a fresh container
docker create --name inspect-tmp <image> && \
docker export inspect-tmp | tar -tf - | grep -E \
  "(^|/)(target|node_modules|\.git|\.m2)/" | head

# 5. Scan for secrets/dirs accidentally baked in
docker history --no-trunc <image> | grep -Ei "COPY|ADD"

# 6. Rebuild with multi-stage + .dockerignore
cat .dockerignore
# .git
# target
# node_modules
# **/.idea
# *.log
# .env
#

docker build -t app:v1.2.0 .

# 7. Optional jlink minimal runtime (saves 150MB+)
FROM eclipse-temurin:21-jre AS jre-build
RUN jlink --add-modules java.base,java.sql,java.naming,java.desktop,java.management \
   --strip-debug --no-man-pages --no-header-files \
   --compress=2 --output /jre
FROM debian:bookworm-slim
COPY --from=jre-build /jre /usr/local/jre
COPY --from=be-build /app/target/app.jar /app.jar
ENTRYPOINT ["/usr/local/jre/bin/java", "-jar", "/app.jar"]

# 8. Verify the new image
docker images app:v1.2.0            # total size
docker history app:v1.2.0 --no-trunc
```

## Root Cause

- **`COPY . .` without `.dockerignore`** — the single highest-impact cause. It copies dev artifacts (`target/`, `node_modules/`, `.git` with its full history, `.idea`, `*.log`, `*.pid`, `*.db`, `.env`) into the build context. `.git` alone routinely adds hundreds of MB.
- **Full JDK at runtime.** `FROM openjdk:11-jdk` and `maven:3.8-jdk-11` ship a complete JDK (compiler, debug symbols, docs) needed at *build* time only. Runtime needs only the JRE → 200MB saved instantly by `eclipse-temurin:21-jre`.
- **Build tools + artifacts in a single stage.** One stage mixing Node, Maven, runtime means a union of *everything*. Multi-stage scoping to "build here, run there" creates low-weight final images.
- **Cache directories inside layers.** `node_modules/.cache`, Maven `.m2`, npm's cache, `apt-get` lists (`rm -rf /var/lib/apt/lists/*`), `npm cache clean --force` — deleted in a later RUN, they *still ship* because layers are immutable.
- **"Fat" JAR bloated by backend static assets duplicates:** frontend `dist/` copied into multiple places (once over `static` in the JAR and again as a separate volume COPY) duplicates MBs.
- **Rebuild invalidation making caching impossible:** naive `COPY . .` at the top of the Dockerfile invalidates every layer downstream on every commit, so the pipeline re-fetches the entire Maven online repo each build → long CI independent of size.

## Immediate Mitigation

1. **Free up short-term storage & speed up CI immediately:**
   ```bash
   docker image prune -a -f          # clean dangling images
   docker container prune -f
   ```
2. **Adopt `.dockerignore` NOW** (5-minute fix, huge win) so the next build excludes `target/`, `node_modules/`, `.git`. Even without restructuring the Dockerfile, this can shrink images by 60–70%.
3. **Switch runtime base from `openjdk:11-jdk` to `eclipse-temurin:21-jre`** (or `17-jre`), keeping `maven`/`node` stages separate — immediate ~300MB savings, no code impact.
4. **Add multi-arch + compression to the pipeline**: switch to `docker buildx build --output=type=registry` with compression `--build-arg BUILDKIT_IMAGE_COMPRESSION=zstd` to cut *compressed* pull size (registry transfer and cold pull time).
5. **Pin the tag that matters**: stop tagging images with ambiguous tags (`latest`) so each pull isn't a huge re-transfer; use `--digest` references in deployment manifests.

While the refactor is in progress, target the *compressed* size for registry costs and the *uncompressed* for node disk — report both.

## Permanent Fix

1. **Multi-stage Dockerfile as canonical** (as in Ideal Answer) with:
   - `node` stage → `dist/` only
   - `maven` stage → `target/app-*.jar` only
   - latest LTS JRE runtime stage, non-root user, no package manager.
2. **Enforce a `.dockerignore` in the repo, reviewed at each PR**, listing `.git`, `target`, `node_modules`, `.idea`, `*.log`, `.env*`, `**/caches`.
3. **Introduce a size budget gate in CI**: fail the build if the resulting compressed image exceeds (e.g.) 300MB. Tooling: `crane digest`/`skopeo inspect` to read compressed size after push, or compare in a build target with `docker images`. This prevents silent regression and makes size a first-class SLO.
4. **Adopt `dive` in CI as an optional analysis job**, at least `dive <image> --ci` with a max efficiency score (e.g., >96%).
5. **Make layer caching deliberate**: order `COPY` operations by change frequency (deps → pom.lock → src), cache `~/.m2` and `~/.npm` through BuildKit cache mounts:
   ```dockerfile
   RUN --mount=type=cache,target=/root/.m2 mvn -DskipTests package
   RUN --mount=type=cache,target=/root/.npm npm ci
   ```
   These cache mounts don't ship in the image and make rebuilds fast.
6. **Consider `jlink`/distroless as the next evolution** when the app is stable and debugging needs are minimal; keep the JRE until then to preserve `jcmd`/`jstack` operations.
7. **Periodic review** (quarterly) of images against the size budget when dependencies/jdks change (a JDK 21 upgrade alone can shift base-image weight 10–20%).

## Monitoring

- **Build-time metrics:**
  - Image size by service/release over time → line chart in the CI dashboard (compressed + uncompressed).
  - Time to push + pull per image → alerts if it exceeds SLO (e.g., 3 min).
- **Runtime metrics:**
  - `container_image_size` (from image registry webhook or `cadvisor`) — track to size-budget.
  - Pod/node disk saturation: `container_fs_usage_bytes` per node; large images cause node disk pressure on rolling updates.
- **Registry metrics:**
  - Registry blob storage per image, per team, per month → cost attribution; the size gate keeps cost predictable.
  - Pull rate through the registry vs bandwidth cost; image pull latency percentiles (p95/p99).
- **Alert examples:**
  - `image_size_bytes > 400MB` → warn; `> budget` → build-fail.
  - Pull duration > 5 min for a critical service → page (cold-start regression).
- **Operational signal:** correlate image size against deployment failure rate (data-heavy images fail more pulls / run out of dir space / hang on node).

## Security

- **Smaller images = smaller attack surface**: fewer packages, no package manager, no compiler toolchain at runtime = fewer exploitable CVE surfaces. Distroless/slim images materially reduce the CVE count (Trivy/anchore scans confirm).
- **The fat image trap**: an image that ships `node_modules` and Maven repos also ships their *known-vulnerable transitive dependencies* — scanning the image surface is not the same as scanning the SBOM. Enforce SBOM generation (`syft`) alongside the size budget.
- **Never strip security automation for size**: size optimizations like `--squash` don't remove need for daily Trivy/Grype scans; and moving to distroless changes *forensics* (no shell), so pair with structured logs and SIEM.
- **Base-image pinning**: pin exact digests (`eclipse-temurin:21-jre@sha256:...`) so "smaller official tag" can't quietly pull a base with new CVEs.
- **Secrets must not be in the build context**: `.dockerignore` prevents `COPY . .` from baking `.env`/credentials; lint for the copy paths. No build secret should end in a layer (`--secret=id=maven_settings, src=...`).

## Production Considerations

- **Reliability**: image size impacts rollout speed and node boot time; a 2.5GB image on a rolling deployment doubles transfer, exhausts node disk and increases interim states. Smaller images → faster, safer rollouts; also reduces kubernetes `ImagePullBackOff` windows.
- **HA**: with a digest-pinned, small image, multiple nodes can pull concurrently without exhausting registry bandwidth; this improves the blast radius of a failed node rebalancing.
- **Scalability**: autoscaling cold starts dominated by image pull time — moving from 2.5GB→250MB is an order-of-magnitude improvement in scale-out latency (and avoid `warm pool` budgets).
- **Cost**: registry storage (GB-month), network egress per pull across nodes, and CI machine minutes are all proportional to image size; size gates are a CFO-friendly optimization.
- **Compliance**: build provenance, digest-pinned base, and immutable tags satisfy SBOM/attestation requirements; distroless+minimal orbits reduce SCRM workload.
- **Operational**: smaller image = faster debug cycles pull-robust fleet; but remove shell/`bash` carefully — some runbook commands (`docker exec bash`) stop working on distroless, which teams must notice during change control.

## Senior-Level Answer

"The image is 2.5GB because the Dockerfile copies the entire build context (`COPY . .` with no `.dockerignore`), which drags in `target/`, `node_modules/`, `.git`, plus a full-JDK base, and caches that never get cleaned because Docker layers are immutable. That's not a Docker problem, that's a Dockerfile design problem. The fix is a multi-stage build: a node stage for the React `dist`, a Maven stage for the JAR, and a JRE-only runtime stage; enforce a `.dockerignore`; mount caches (`--mount=type=cache`) instead of copying build state; and add a size budget gate in CI so it never regresses. For a Spring Boot + React app this lands ~250MB (or ~130MB with `jlink`), makes pulls single-digit seconds, and cuts both CI time and registry spend. The layer-cache mistrust is the one real gotcha — a 1.5GB lower layer can't be 'deleted' by a later RUN, so measure with `dive`/`docker history` and always sprint to a multi-stage design."

## Architect-Level Answer

"Image size is a *platform cost and reliability metric*, not a developer concern. At the architecture level, I'd treat the image as a regulated supply-chain artifact with explicit size, SBOM, and provenance budgets enforced in CI — because every MB of image shows up four times *multiplied by fleet size*: registry storage, per-node disk, per-pull egress bandwidth, and scale-out latency. So I serialize the changes: first a mandatory `.dockerignore` (the zero-effort 60% win), then multi-stage Dockerfiles owned by a platform golden-pattern template, then the size gate (build-fail beyond budget) plus SBOM attestation and digest-pinning of golden bases.

Architecturally for this Java+React stack I would consider splitting the delivery models: the React app serves best as a next.js/nginx static artifact (independent scaling and CDN), while the Spring Boot service shrinks via `jlink`/GraalVM for its payment-critical endpoints — but only where instrumentation (jvm metrics, openTelemetry) can survive the smaller runtime. The strategic point: image, SBOM, and cache policy are three separated concerns that must be owned at platform level; developer-owned Dockerfiles are the recurring source of 'fat image' debt, and eliminating that debt is a platform-level product initiative with measurable SLOs (build duration, pull latency, registry cost) tracked across quarters."

## Follow-Up Questions

1. Your organization says "we need the full JDK at runtime because we dynamically compile Java at runtime (spring-dev-tools / JSPs / JavaCompiler)." What impact does that have on image size, and what are the alternatives (including sidecar/toolchain containers)?
2. `jlink` creates a *custom-fitted* JRE that is 10x smaller, but many Spring Boot apps fail at runtime with `java.lang.module` or missing `--add-modules` errors. How do you discover exactly which modules your app uses, and how do you keep this reproducible across Spring Boot upgrades?
3. Because Docker layers are immutable, a `RUN rm -rf /var/lib/apt/lists` won't remove the files from the earlier layer — they're still shipped. How do you *actually* get rid of that space, and how does `--squash` interact with layer-svg attribution?
4. Your multi-stage image is 230MB but *uncompressed*, and the registry charges you for *compressed* size. What does BuildKit's `--output=type=registry` compression settings (`gzip` vs `zstd` vs `zstd:fastest`) do to the blob size and to per-node decompression CPU during pull?
5. The developer says "let's just put the 2.5GB image in a private ACR/ECR, the bandwidth is cheap." As an architect, walk me through the *serious* operational and security costs of a fat image besides the raw dollar figure (think cold-start SLOs, node evictions, and multi-region sync).