# 31. Artifact Versioning Conflict in Deployment

## Scenario

It's 3:45 PM on a Friday. Team A deployed their microservice `order-service` version `2.3.1` to production at 3:30 PM. At 3:40 PM, Team B deployed the same `order-service` but their CI pipeline pulled version `2.3.0` from a stale cache instead of building the latest code. The production environment now has a mix of pods: some running `2.3.1` (with the new payment gateway integration) and others running `2.3.0` (without it). Users are reporting intermittent failures — about 30% of payment requests fail because they hit pods running the old version that doesn't support the new payment gateway protocol. The load balancer is distributing traffic across both versions. The on-call engineer sees version mismatch in pod labels but isn't sure which version is "correct."

## Interviewer Question

"How do you resolve this immediate version conflict in production, and what CI/CD architecture changes would you implement to prevent artifact versioning conflicts from ever happening again?"

## What I Should Think About

- Immediate production impact: 30% of payments are failing — this is a P1 incident
- Need to determine which version should be running (answer: 2.3.1, the intended version)
- The root cause is artifact cache invalidation failure in Team B's CI pipeline
- Consider semantic versioning, immutable artifacts, and artifact repository policies
- Think about deployment locking mechanisms
- Consider how this relates to monorepo vs polyrepo strategies
- Need to address both the immediate fix and the systemic prevention

## Ideal Answer

**Immediate Resolution (First 10 minutes)**

1. **Determine the correct version**: Team A's intent was to deploy 2.3.1. Team B accidentally deployed 2.3.0. The correct version is 2.3.1.
2. **Scale up 2.3.1 pods**: Increase replicas running 2.3.1 to handle all traffic
3. **Scale down 2.3.0 pods**: Reduce 2.3.0 replicas to 0
4. **Verify**: Confirm all pods are now running 2.3.1 and payment success rate returns to 100%

```bash
# Scale up the correct version
kubectl scale deployment order-service --replicas=10 --record \
  -n production

# Check which pods are running which version
kubectl get pods -n production -l app=order-service \
  -o custom-columns=POD:.metadata.name,VERSION:.metadata.labels.version

# Terminate old version pods
kubectl delete pods -l app=order-service,version=2.3.0 -n production
```

**Root Cause Analysis**

Team B's CI pipeline had a Docker cache hit on the base image that included the old artifact. The build cache wasn't properly invalidated because:
- The Dockerfile used `COPY . .` without proper cache-busting
- The CI pipeline didn't use `--no-cache` for production builds
- There was no artifact version verification step in the deployment pipeline

**Systemic Prevention**

1. **Immutable artifacts with unique tags**: Never use `latest` or allow cache-based artifact reuse
2. **Artifact provenance tracking**: Every artifact in the registry must trace back to a specific commit SHA
3. **Deployment pipeline validation**: Before deploying, verify the artifact's build metadata matches the expected commit
4. **Environment lock during deployments**: Prevent concurrent deployments to the same environment
5. **Artifact promotion pipeline**: Artifacts flow DEV → STAGING → PROD without rebuilding

## Architecture

```
CONFLICT SCENARIO:
┌─────────────────────────────────────────────────────┐
│              Production Cluster                      │
│                                                      │
│  ┌──────────────┐  ┌──────────────┐                 │
│  │ Pod A (v2.3.1)│  │ Pod B (v2.3.0)│ ← WRONG      │
│  │ Payment: New  │  │ Payment: Old  │                │
│  └──────────────┘  └──────────────┘                 │
│  ┌──────────────┐  ┌──────────────┐                 │
│  │ Pod C (v2.3.1)│  │ Pod D (v2.3.0)│ ← WRONG      │
│  │ Payment: New  │  │ Payment: Old  │                │
│  └──────────────┘  └──────────────┘                 │
│                                                      │
│  Load Balancer → 50% hit old version → 30% failures │
└─────────────────────────────────────────────────────┘

PREVENTION ARCHITECTURE:
┌──────────┐    ┌───────────┐    ┌──────────────┐
│  Git Push │───→│    CI     │───→│ Build Artifact│
│  (commit  │    │  Build    │    │ Tag: commit-  │
│   SHA)    │    │ (no cache)│    │ sha + version │
└──────────┘    └───────────┘    └──────┬───────┘
                                        │
                                  ┌─────┴─────┐
                                  │  Registry  │
                                  │ (immutable)│
                                  └─────┬─────┘
                                        │
                              ┌─────────┴─────────┐
                              │ Deploy Pipeline    │
                              │ 1. Verify artifact │
                              │    metadata matches│
                              │    expected commit  │
                              │ 2. Acquire env lock│
                              │ 3. Deploy          │
                              │ 4. Release lock    │
                              └───────────────────┘
```

## Investigation

1. **Check current pod versions**: `kubectl get pods -o json | jq '.items[] | {name: .metadata.name, version: .metadata.labels.version}'`
2. **Verify artifact tags in registry**: Check ECR/DockerHub for what tags exist and their digests
3. **Review CI build logs**: Check if Team B's build actually compiled the code or used a cached image
4. **Compare image digests**: The correct version should match Team A's image digest
5. **Check deployment history**: `kubectl rollout history deployment/order-service -n production`
6. **Review the artifact repository**: Check when each version was pushed and by which pipeline
7. **Trace the lineage**: Which Git commit does each artifact correspond to?
8. **Check for deployment locks**: Was there any mechanism to prevent concurrent deployments?

## Commands

```bash
# Identify version mismatch
kubectl get pods -n production -l app=order-service \
  -o custom-columns=POD:.metadata.name,VERSION:.metadata.labels.version,STATUS:.status.phase

# Check image digests for each version
kubectl get pods -n production -l app=order-service \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\n"}{end}'

# Compare with registry
aws ecr describe-images --repository-name order-service \
  --query 'imageDetails[*].{tag:imageTags,digest:imageDigest, pushed:imagePushedAt}'

# Immediate fix: Scale to correct version
kubectl set image deployment/order-service \
  order-service=123456789.dkr.ecr.us-east-1.amazonaws.com/order-service:2.3.1 \
  -n production

# Check deployment rollout history
kubectl rollout history deployment/order-service -n production

# Verify deployment revision details
kubectl rollout history deployment/order-service -n production --revision=12

# Check which commits correspond to which versions
git log --oneline --decorate | head -20

# Audit artifact provenance in registry
aws ecr batch-get-image --repository-name order-service \
  --image-ids imageTag=2.3.1 --query 'images[].imageManifest' \
  --output text | jq '.config.label'
```

## Root Cause

| Root Cause | How to Prevent |
|---|---|
| Docker build cache serving stale artifacts | Use `--no-cache` or cache-from with digest-based keys |
| No artifact metadata verification at deploy time | Add commit SHA validation step before deployment |
| Concurrent deployments to same environment | Implement deployment locks (Redis lock, K8s leader election) |
| Version tags not tied to Git commits | Enforce artifact tagging convention: `git-sha-version` |
| No artifact immutability policy | Configure registry to reject overwriting tags |
| CI pipeline not recording build metadata | Store build provenance (commit, branch, build ID) as OCI annotations |

## Immediate Mitigation

1. **Determine the correct version** (2.3.1) by checking Team A's deployment intent
2. **Force all pods to run the correct version**: `kubectl set image deployment/order-service order-service=<correct-image>:2.3.1`
3. **Verify all pods are updated**: Check pod list for version labels
4. **Monitor payment success rate** — should return to ~100%
5. **Notify the team**: Communicate what happened and that it's resolved
6. **Preserve evidence**: Screenshot pod versions, save CI logs, save registry audit logs

## Permanent Fix

1. **Immutable artifact tags**: Configure ECR/DockerHub to prevent tag mutation (once `2.3.1` is pushed, it cannot be overwritten)
2. **Deploy-time artifact verification**: Add a step that compares the image digest being deployed against the digest recorded in the CI build
3. **Deployment locking**: Use a distributed lock (Redis, Consul, or K8s annotations) to serialize deployments to the same environment
4. **Build metadata in artifacts**: Use OCI image annotations to embed commit SHA, branch, build number in every image
5. **Artifact promotion pipeline**: Build once in CI, promote through environments by re-tagging (never rebuild)
6. **Deploy dashboard**: Real-time visibility into what version is running in each environment
7. **Branch protection**: Require PR reviews for any pipeline change that affects artifact building

## Monitoring

- **Alert when pod versions don't match**: Compare pod labels against expected deployment version
- **Track deployment concurrency**: Alert if two deployments to the same environment start within 5 minutes
- **Monitor artifact age**: Alert if an artifact in production was built more than X days ago
- **Version consistency check**: Periodic audit comparing deployed versions across pods
- **Payment success rate**: Primary business metric affected by this issue
- **Deployment frequency and success rate**: Track DORA metrics

## Security

- Immutable tags prevent supply chain attacks where an attacker pushes a malicious image with an existing tag
- Artifact signing (cosign/Notation) ensures images haven't been tampered with between build and deploy
- Build provenance (SLSA framework) provides cryptographic verification of what code produced the artifact
- Registry access controls prevent unauthorized image pushes
- Audit logs for all image push/pull operations

## Production Considerations

- **Reliability**: Version conflicts can cause cascading failures in dependent services
- **Scalability**: Deployment locks must be lightweight and not become bottlenecks
- **Cost**: Immutable tags increase storage costs; implement lifecycle policies for old versions
- **Compliance**: Regulatory environments may require full artifact provenance and audit trails
- **Operational**: Runbook for version conflict incidents should be in the incident response playbook
- **HA**: Deployment locking must handle leader failures gracefully (auto-release after timeout)

## Senior-Level Answer

"First, I'd resolve the immediate conflict by forcing all pods to run version 2.3.1 — the version Team A intended. I'd use `kubectl set image` to update the deployment and verify all pods are running the correct version. Then for prevention: I'd implement immutable artifact tags in ECR (prevent tag overwrite), add a deploy-time verification step that validates the image digest matches the CI build's recorded digest, and implement deployment locks using Redis to prevent concurrent deployments to the same environment. Every artifact would carry build metadata (commit SHA, branch, build ID) as OCI annotations, enabling full traceability from production pod back to source commit."

## Architect-Level Answer

"This incident reveals three architectural gaps. First, **artifact immutability** — I'd implement a policy where once an image tag is pushed, it's immutable. Tags follow the convention `commit-sha-short-version` (e.g., `a1b2c3d-v2.3.1`). Second, **deployment orchestration** — I'd implement a centralized deployment manager (using ArgoCD with sync windows, or a custom controller) that serializes deployments to the same environment and provides a deployment calendar. Third, **artifact provenance** — following SLSA Level 3 requirements, every artifact would have signed build metadata proving it was built from a specific commit by a specific pipeline, and the deployment pipeline cryptographically verifies this before deploying. At the enterprise level, I'd establish an **artifact governance policy** across all 15 teams, with a shared library for artifact tagging and a centralized registry with retention policies. The goal is that it becomes architecturally impossible to deploy a mismatched version."

## Follow-Up Questions

1. "How would you implement deployment locking across 50 microservices that share the same environment?"
2. "Your immutable tag policy conflicts with the `latest` tag convention some teams use. How do you handle this?"
3. "How do you handle artifact versioning in a monorepo where multiple services share a library?"
4. "What if the version conflict was in a stateful service with persistent data? How does that change the resolution?"
5. "How do you implement artifact promotion (build once, deploy everywhere) when different environments need different build configurations?"
