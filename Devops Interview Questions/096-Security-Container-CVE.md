# 96. Critical CVE Found in Production Container Image

## Scenario

A critical CVE (CVSS 9.8) was discovered in the base image used by ALL production containers. The CVE allows Remote Code Execution (RCE). You have 400 running containers across 200 pods in Kubernetes. The vulnerable package is in the OS base image (e.g., `glibc`/OpenSSL-style flaw). Trivy/Anchore scan flagged the specific image `app:2.4.0`. A patch release image (`app:2.4.1`) is available but you haven't tested it. Traffic is heavy. You cannot take downtime. You need to remediate without disrupting the service.

## Interviewer Question

"A critical CVE (CVSS 9.8) with RCE was found in the base image of all production containers. You have 400 containers across 200 pods. How do you remediate without downtime?"

## What I Should Think About

- Severity and rush vs risk of breaking an untested image
- Immediate: verify exploitability from within the cluster (is it reachable externally?)
- Can we use network controls to buy time (block ingress/egress to affected servces)
- Rollout strategy: build patched image, test in staging, roll out carefully (rolling update, canary, blue-green)
- Kubernetes strategies: `maxUnavailable: 0`, `maxSurge`, pod disruption budget (PDB)
- Rolling update vs replace: image update in Deployment spec triggers rollout
- Monitoring during replacement; readiness/liveness probes
- If service mesh in place: add per-pod routing for canary
- Vulnerability management pipeline: CI image scanning gate (build-time), admission webhook
- SBOM — attach, so base image changes are tracked
- Runtime detection: Falco/GKE/Karpenter events; CVE exploit attempt detection

## Ideal Answer

**Immediate (minutes):**
1. Confirm perimetral exposure: is the vulnerable container reachable from the internet? Apply a NetworkPolicy to allow only the intended ingress/egress from the affected workload — closes the direct RCE path while you patch
2. Confirm no active exploit (look in audit logs/Falco for suspicious commands, reverse shells)

**Build (hour 1-2):**
3. Build `app:2.4.1` from the patched base image (or rebuild with `apt-get upgrade`/`yum update` on the base)
4. Run the scan again (Trivy) → confirm CVE gone
5. Deploy to staging, run the smoke/integration suite + performance sanity

**Rollout (hour 2-3):**
6. Use a rolling update with a Pod Disruption Budget; safe settings: `maxUnavailable: 0`, `maxSurge: 25%`
7. Start canary: update a small deployment (10%) → verify health → roll rest
8. Monitor: pod restarts, error rate, 5xx, p99 latency, resource usage during rollout

**Follow-up:**
9. Add registries/CI gating: fail builds that pull a vulnerable base; add admission webhook (e.g., OPA/Gatekeeper or Kyverno) to reject workloads with known vulns
10. Maintain SBOM + track base image updates; automation to rebuild images in response to upstream CVE advisories

## Architecture

```
  K8S Rollout (zero-downtime) for patched image:
  ┌─────────────────────────────────────────────────────────────┐
  │ Deployment app (replicas: 200)                              │
  │ strategy:                                                   │
  │   rollingUpdate:                                            │
  │     maxSurge: 25%      (new ready pods first)               │
  │     maxUnavailable: 0  (never below capacity)               │
  │                                                             │
  │ Before:  [old:2.4.0] x 200  (vulnerable)                   │
  │ Step 1:  old 200 + surge 50 new [2.4.1]  → 250 total        │
  │ Step 2:  old 150, new 100   (old drained, new kept ready)   │
  │ Step 3:  old 0, new 200 (2.4.1, patched)                   │
  └─────────────────────────────────────────────────────────────┘

  Containment network layer (during build time):
  ┌──────────────┐   ┌───────────────────────────────┐
  │ ingress LB   │──▶│ NetworkPolicy (allow only)     │
  │ only 443/80  │   │  pod=app from=ingress          │
  └──────────────┘   │  egress: only api.example.com  │
                     └───────────────────────────────┘

  CI/CD vulnerability gate:
  git commit → build → JFrog/ECR scan (trivy) → FAIL if vuln=critical
                  → admission webhook (Kyverno) → reject pod if CVE
                  → deploy
```

## Investigation

**Step 1: Confirm vector & exposure**
```bash
# But first: is the CVE actually exploitable from here?
# If OpenSSL RCE → is TLS terminated at the pod or LB?
# If glibc/sshd → is the service port exposed?
kubectl get svc -A | grep app
kubectl get networkpolicies -A
kubectl get endpoints app
```

**Step 2: Validate the vulnerable image version**
```bash
kubectl get deploy app -o jsonpath='{.spec.template.spec.containers[0].image}'
# → my.registry.com/app:2.4.0

trivy image --severity CRITICAL,HIGH my.registry.com/app:2.4.0 \
  --exit-code 1 --no-progress
```

**Step 3: Confirm patch availability**
```bash
# Pull latest base image and compare:
docker pull base:latest
trivy image --severity CRITICAL base:latest --format json | jq .Results
# Verify base:latest is clear, or build from distroless / patched tag
```

**Step 4: Check for active exploitation (runtime)**
```bash
# Falco event stream
falco --priority warning
# check audit logs / CRI events
journalctl -b | grep -i "exec\|proc_creation"
# look for outbound connections from pods
kubectl logs -l app=app --tail=50 | grep -iE "curl|wget|nc -e|/bin/sh"
```

**Step 5: Assess blast radius of base image**
```bash
# What OTHER images built on this base exist in our registry?
# → they're ALL vulnerable → plan separate rebuilds
# e.g., check ImageStream / ECR repos WHERE created-from same base
```
**Step 6: Assess dependencies shipped in the running container**
```bash
# runtime SBOM:
trivy sbom my.registry.com/app:2.4.0 --format spdx-json
```

## Commands

```bash
# BUILD patched image with locked deps
docker build -t my.registry.com/app:2.4.1 .
docker push my.registry.com/app:2.4.1

# scan patched image
trivy image --severity CRITICAL,HIGH \
  --ignore-unfixed my.registry.com/app:2.4.1

# roll out with zero downtime (deployment spec):
kubectl set image deploy/app app=my.registry.com/app:2.4.1
kubectl rollout status deploy/app --timeout=300s

# if you want a canary deployment in parallel
kubectl create deploy app-canary --image=...:2.4.1 --replicas=20
# route 5% via service mesh (Istio VirtualService weight)
kubectl apply -f vs-weight-95-5.yaml

# protect against traffic during drain
kubectl create -f - << 'EOF'
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: {name: app-pdb}
spec:
  minAvailable: 190
  selector: {matchLabels: {app: app}}
EOF

# verify rollout
kubectl get pods -l app=app -o wide
kubectl get events --field-selector type=Warning | tail
kubectl top pods -l app=app
```

## Root Cause

Not a single incident — a systemic gap:
1. **No build-time vulnerability gate** in CI: vulnerable base got through and deployed
2. **Base image not lifecycle-managed**: nobody tracked upstream base updates; drift snowballed
3. **No SBOM** to map which images shared the vulnerable layer
4. **Production rollout lacked canary/zero-downtime levers** (fixed by PDB + surge)
5. **Registry not auto-patched** — no automated rebuild pipeline tied to CVE feeds

## Immediate Mitigation

```bash
# 1. Contain: block egress/ingress network to the vulnerable workload
kubectl apply -f - << 'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: app-lockdown}
spec:
  podSelector: {matchLabels: {app: app}}
  policyTypes: [Ingress, Egress]
  ingress:
    - from: [{namespaceSelector: {matchLabels: {role: ingress}}}]
  egress:
    - to: [{ipBlock: {cidr: 10.0.0.0/8}}]  # internal only
EOF

# 2. Patch image via safe rollout
kubectl set image deploy/app app=...:2.4.1
kubectl rollout status deploy/app

# 3. Check for exploitation: Falco / audit
falco --priority warning | grep -iE "exec|shell|reverse"

# 4. Block failed-pod creation with admission (stop future bad launches)
```

## Permanent Fix

1. **CI/CD image scan gate**: Trivy/Anchore/Sysdig in CI; fail pipeline on criticals
2. **Admission webhook** (Kyverno/OPA) rejecting pods with known critical images
3. **Base image lifecycle automation**:
   - Chain builds of downstream image on base update (CI triggers)
   - Track base image ENV + SBOM in the registry
4. **Image maturity**: pull base images through a private mirror, pin base tags, upgrade cadence
5. **Runtime security** (Falco) detection of post-exploit activity
6. **SBOM + CVE watch**: subscribe to security advisories; automated rebuild pipeline

## Monitoring

```bash
# Prometheus/Grafana:
# - image vulnerability count per image (via trivy/JFrog xray updates)
# - % pods running outdated / last scanned image
# - base image release cadence & drift
# - image scanning failures in CI/CD pipeline

# Runtime:
# - Falco: execve into containers, reverse shell, /proc access
# - Kube events: OOMKilled/ImagePull errors during rollout
# - Gatekeeper audit: constraints rejected pods with CVEs

# Alerts:
# - New CRITICAL CVE in a running image → P1 page
# - Pods using image not scanned within X days → warning
# - image pull/digest change (out-of-band deploy) → audit alert
```

## Security

- Segregate the registry (private), control push with image signing/attestation (cosign)
- Admission policy enforcing `imagePullPolicy: IfNotPresent` + signer verification
- Network: default deny, allowlist egress to reduce exploit impact
- Run containers as non-root, read-only rootfs (defense-in-depth beyond patching)
- Distro: prefer distroless/minimal base (smaller attack surface → fewer CVEs)

## Production Considerations

- **HA**: zero-downtime rollout relies on readiness probes being accurate — make sure `/health` reflects actual readiness
- **Reliability**: rolling update with `maxUnavailable: 0` is safest for latency-sensitive apps
- **Cost**: distroless/minimal images cut CVE surface and registry storage
- **Compliance**: organizations require patching within SLA (e.g., 7d for CRITICAL) — measure compliance as an SLO
- **Operational**: prebuilt standby patched images for infra base images; freeze upgrades during peak windows
- **Governance**: single source of truth for approved base images (organization-level golden image policy)

## Senior-Level Answer

"First, I'd contain the window: tighten NetworkPolicy so the vulnerable workload can't be reached or call out, and confirm via Falco/audit logs there's no existing exploitation. Then build and scan the patched image, run it through staging, and roll it out with a PodDisruptionBudget and a rolling update (`maxUnavailable: 0`, `maxSurge: 25%`), canarying 5-10% first to watch for errors. The sustain fix is a CI image-scan gate (Trivy/Anchore) that fails the build on critical CVEs, an admission webhook rejecting known-vulnerable images, base image lifecycle automation with SBOM tracking, and Falco runtime detection. The independence of the patch + controlled rollout both happen without downtime."

## Architect-Level Answer

"A critical CVE forces us past 'patch and hope' into supply-chain security. The architecture should make vulnerability response nearly automatic: build-time scanning gates, signed images enforced at admission, and a base-image pipeline where updating a base tag triggers rebuild of all downstream images. I'd add SBOM generation and CVE-mapping so we know the blast radius instantly. Canonically, mitigation needs network-zero-training: default-deny NetworkPolicies and runtime Falco rules so an unpatched container is limited even if exploited. Rollout-wise, a standardized zero-downtime playbook with PDB + canary + `maxUnavailable:0` gains should be the default for all workloads. And we treat CVE SLA as a compliance metric: 7 days for criticals measured and reported. That turns the firefight into a predictable process."

## Follow-Up Questions

1. "If the patched image fails staging tests, what ops do you run to reduce the CVE window while you wait?"
2. "How do you enforce 'no vulnerable image gets scheduled' after a base image CVE is disclosed — what tools, and what are their limitations?"
3. "Design the automated pipeline that rebuilds and redeploys your images when a base image update lands. How do you avoid broken deploys?"
4. "What's your strategy for the OTHER images built on the same vulnerable base that you didn't know about (SBOM vs registry scan)?"
5. "If exploitation was already detected in one pod, how do you isolate, contain, and evict without taking the whole deployment down?"