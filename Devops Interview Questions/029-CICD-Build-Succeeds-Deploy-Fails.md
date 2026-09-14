# 29. Build Succeeds but Production Deployment Fails

## Scenario

Your team pushes a new feature to main. The CI pipeline runs green — linting passes, all 3,200 unit tests pass, integration tests pass, SonarQube quality gate passes, Trivy finds no critical CVEs, and the Docker image is built and pushed to AWS ECR successfully. The CD pipeline (ArgoCD) picks up the new image tag and begins rolling out to the production Kubernetes cluster. After 5 minutes, the rollout hits its deadline and times out. The pods are in `CrashLoopBackOff`. The application was working perfectly yesterday with the previous version. The on-call engineer pages you at 2 AM. Production traffic is still being served by the old pods (since the rollout never completed), but you cannot deploy anything new.

## Interviewer Question

"The build pipeline is fully green — all stages passed. But the Kubernetes rollout is timing out and pods are crash-looping. Walk me through how you'd troubleshoot and resolve this, step by step, under production pressure."

## What I Should Think About

- Build success does NOT guarantee deploy success — the environments differ
- Immediate priority: production is still running (old pods not terminated yet due to rolling update strategy), so there's no outage yet, but you're stuck
- Need to determine if this is a K8s manifest issue, image issue, configuration issue, or runtime dependency issue
- Must investigate pod logs, events, and container exit codes
- Consider: resource limits, secrets/configmaps, environment variables, readiness probes, liveness probes, network policies
- Rollback should be the first recovery action, root cause analysis comes after
- Need to think about why CI didn't catch this (missing staging environment parity)

## Ideal Answer

**Step 1 — Assess and Stabilize (2 minutes)**

First, confirm production is still serving traffic. Since the rollout never completed, old pods are still running. Verify with `kubectl get pods` — you should see old pods `Running` and new pods `CrashLoopBackOff` or `Error`.

If the new pods are somehow killing old ones (e.g., DaemonSet or sidecar), immediately rollback:
```bash
kubectl rollout undo deployment/myapp -n production
```

**Step 2 — Investigate the Failed Pods**

Check the pod status and exit codes:
```bash
kubectl describe pod <new-pod-name> -n production
kubectl logs <new-pod-name> -n production --previous
kubectl logs <new-pod-name> -n production -c <init-container>
```

Common findings:
- Exit code 1: Application error (check logs)
- Exit code 137: OOMKilled (memory limit too low)
- Exit code 139: Segfault
- Init container failure: ConfigMap/Secret not mounted
- ImagePullBackOff: Registry credentials or image tag issue

**Step 3 — Compare Environments**

The most common root cause: the CI test environment and production differ. Check:
- Environment variables (production has different DB_HOST, different secrets)
- ConfigMaps and Secrets (production has different values)
- Resource limits (production is more constrained)
- Network policies (production blocks certain egress)
- Service accounts and RBAC permissions

**Step 4 — Fix and Redeploy**

Based on root cause, apply the fix. Most common real-world scenarios:

1. **Missing Secret/ConfigMap**: The new version requires a new env var that was added to dev/staging but never created in production
2. **OOMKilled**: Production traffic patterns cause higher memory usage than CI test traffic
3. **New dependency not available**: The app tries to connect to a new service that doesn't exist in production yet
4. **Probes misconfigured**: The new code changes the startup timing, causing the liveness probe to kill pods before they're ready

**Step 5 — Validate and Document**

After successful deployment, update the CI pipeline to catch this class of issue (evidentiary gap closure).

## Architecture

```
Developer Push → Git → CI Pipeline (Jenkins/GitHub Actions) → Green ✓
                                                      ↓
                                              Build Docker Image
                                                      ↓
                                              Push to ECR (✓)
                                                      ↓
                                              ArgoCD Sync Trigger
                                                      ↓
                                         Kubernetes Cluster (Production)
                                                      ↓
                                    ┌─────────────────────────────┐
                                    │  Deployment Rollout         │
                                    │  Strategy: RollingUpdate    │
                                    │                             │
                                    │  Old Pods: Running ✓        │
                                    │  New Pods: CrashLoop ✗      │
                                    │  (rollout stuck at 50%)     │
                                    └─────────────────────────────┘
```

## Investigation

1. Check rollout status: `kubectl rollout status deployment/myapp -n production`
2. List pods and filter by status: `kubectl get pods -n production -l app=myapp`
3. Describe failing pod for events: `kubectl describe pod <pod-name> -n production`
4. Check container logs (current and previous): `kubectl logs <pod-name> --previous`
5. Check init container logs if applicable: `kubectl logs <pod-name> -c <init-container>`
6. Compare the new vs old pod specs: `kubectl get pod <old-pod> -o yaml` vs `kubectl get pod <new-pod> -o yaml`
7. Verify Secrets and ConfigMaps exist: `kubectl get secrets -n production` / `kubectl get configmaps -n production`
8. Check resource consumption: `kubectl top pod -n production`
9. Review recent changes to the K8s manifests in Git
10. Check if the image was actually pushed to registry: `aws ecr describe-images --repository-name myapp`

## Commands

```bash
# Check current rollout status
kubectl rollout status deployment/myapp -n production

# Immediate rollback if needed
kubectl rollout undo deployment/myapp -n production
kubectl rollout undo deployment/myapp -n production --to-revision=5  # specific revision

# Investigate failing pods
kubectl get pods -n production -l app=myapp -o wide
kubectl describe pod myapp-7d8f9b6c5-x2k4j -n production
kubectl logs myapp-7d8f9b6c5-x2k4j -n production --previous
kubectl logs myapp-7d8f9b6c5-x2k4j -n production -c sidecar

# Check container exit codes
kubectl get pod myapp-7d8f9b6c5-x2k4j -o jsonpath='{.status.containerStatuses[0].lastState.terminated.exitCode}'

# Compare deployments between environments
kubectl diff -f deployment.yaml -n production

# Check if secrets exist
kubectl get secret myapp-db-secret -n production -o jsonpath='{.data.DB_HOST}' | base64 -d

# Verify image in registry
aws ecr describe-images --repository-name myapp --query 'sort_by(imageDetails,&imagePushedAt)[-1]'

# Check events for the namespace
kubectl get events -n production --sort-by='.lastTimestamp' | tail -20

# Force rollback to known good version
kubectl set image deployment/myapp myapp=myapp:2.4.7 -n production
```

## Root Cause

| Root Cause | How to Identify | How to Eliminate |
|---|---|---|
| Missing Secret/ConfigMap in production | Pod events show "secret not found" | Automate secret sync across environments in CI |
| OOMKilled | `kubectl describe` shows `reason: OOMKilled` | Set production-appropriate resource limits; add memory profiling in CI |
| New environment variable missing | App logs show "undefined variable" or connection errors | Use a shared env template; diff env configs between environments in pipeline |
| Liveness probe too aggressive | Pod logs show "shutting down" during startup | Tune probe timing; use startup probes for slow-starting apps |
| Database migration not run | App logs show SQL errors | Add migration step to CD pipeline before deployment |
| Network policy blocking traffic | Pod logs show connection timeouts | Version control network policies; apply them before deployment |
| RBAC permission denied | Pod logs show 403 Forbidden | Audit service account permissions in CI |

## Immediate Mitigation

1. **Rollback immediately** — `kubectl rollout undo deployment/myapp -n production`
2. **Verify old pods are serving traffic** — `kubectl get endpoints myapp -n production`
3. **Notify stakeholders** — Send a message to the incident channel: "Deployment rolled back, production serving previous version, investigating root cause"
4. **Do NOT delete the failed pods yet** — They contain logs needed for investigation
5. **Freeze new deployments** — Prevent other teams from deploying until root cause is understood

## Permanent Fix

1. **Add a pre-deployment validation stage** that runs a subset of tests against a production-like environment (canary or staging that mirrors prod)
2. **Implement environment parity checks** in CI — compare env vars, secrets, resource quotas between staging and production
3. **Add a configuration diff tool** — Before deploying, the pipeline outputs a diff of all environment-specific configurations
4. **Implement deploy-time health checks** — ArgoCD should have `spec.strategy.rollingUpdate.maxUnavailable: 0` and `maxSurge: 1` with proper timeouts
5. **Add a pre-sync hook** in ArgoCD that validates all required secrets/configmaps exist
6. **Standardize environment promotion** — All environment-specific configs should be managed in Git with automated validation
7. **Add integration tests that run against production-like environments** with production-like data shapes

## Monitoring

- **Alert on `kube_deployment_status_replicas_available` < desired** for > 2 minutes
- **Alert on `kube_pod_container_status_restarts_total`** increasing
- **Monitor `kubectl rollout status`** as a pipeline health check
- **Track deployment frequency and failure rate** with DORA metrics
- **Alert on OOMKilled events**: `kube_pod_container_status_last_terminated_reason{reason="OOMKilled"}`
- **Dashboard for deployment pipeline stages** with pass/fail rates per stage
- **Log aggregation** for pod events with alerting on CrashLoopBackOff

## Security

- Ensure production secrets are not the same as staging/dev
- Secret rotation after any potential exposure during debugging
- RBAC should limit who can `kubectl describe` secrets in production
- Audit logs for all kubectl commands against production cluster
- Consider sealed-secrets or external-secrets-operator for GitOps-safe secret management
- Ensure the CI service account has least-privilege permissions on the production cluster

## Production Considerations

- **High Availability**: Rolling update strategy with `maxUnavailable: 0` ensures zero downtime during failed deployments
- **Scalability**: Pre-deployment validation should run in parallel with environment provisioning
- **Reliability**: Canary deployments (deploy to 5% traffic first) catch issues before full rollout
- **Cost**: Running a production-mirroring staging environment is expensive but prevents these incidents
- **Compliance**: All deployment events should be auditable; maintain deployment logs for 90 days
- **Operational**: Runbooks for deployment failures should be accessible and tested quarterly

## Senior-Level Answer

"First, I'd confirm production is still healthy — since we use rolling updates with `maxUnavailable: 0`, old pods should still be serving. Then I'd check pod logs and events for the crash reason. Most commonly, this is a missing environment variable or secret that exists in staging but not production. I'd rollback immediately with `kubectl rollout undo`, then investigate the root cause by comparing pod specs between old and new versions and checking container exit codes. The systemic fix is adding an environment-parity validation step to the CI pipeline that diffs critical configs between staging and production before promoting a deployment, plus implementing canary deployments so we catch issues with a small traffic percentage before full rollout."

## Architect-Level Answer

"This incident reveals a gap in our deployment pipeline's validation strategy. At the architecture level, I'd implement three layers of defense: First, a **pre-deployment validation stage** in the pipeline that programmatically compares environment configurations (secrets, configmaps, resource quotas, network policies) between staging and production, failing the pipeline if critical mismatches exist. Second, a **canary deployment strategy** using Argo Rollouts or Flagger that gradually shifts traffic from 0% to 5% to 25% to 100%, with automated rollback if error rates spike. Third, **environment-as-code** where all environment-specific configurations are version-controlled with automated drift detection. I'd also invest in a production-mirror staging environment provisioned via IaC to maximize CI fidelity. The long-term goal is that the CI pipeline tests against an environment that is indistinguishable from production, making this class of failure impossible."

## Follow-Up Questions

1. "Your rollback succeeded but you notice in the logs that some users received errors during the brief period when new pods were starting. How do you handle the partial impact?"
2. "How would you implement canary deployments with automatic rollback using Argo Rollouts?"
3. "What if the failure was caused by a database migration that already partially ran? How does that change your rollback strategy?"
4. "How do you ensure environment parity between staging and production without giving developers access to production infrastructure?"
5. "If this keeps happening — builds pass in CI but fail in production — what architectural changes would you propose for the entire CI/CD pipeline?"
