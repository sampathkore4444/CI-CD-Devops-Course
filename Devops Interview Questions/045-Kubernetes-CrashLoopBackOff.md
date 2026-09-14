# 45. Production Pod in CrashLoopBackOff State

## Scenario

It's 2:47 AM on a Saturday. PagerDuty fires an alert: the `order-service` pod in the `production` namespace is in `CrashLoopBackOff`. This Java Spring Boot microservice handles all e-commerce order processing. The pod has restarted 12 times in the last 6 minutes. Users are seeing HTTP 500 errors on the checkout page. The deployment has 3 replicas but only 1 is healthy. The other 2 are stuck in CrashLoopBackOff. Revenue is being impacted every minute this continues.

The application was deployed 30 minutes ago as part of a routine release. The staging environment passed all tests. The container image `registry.internal/order-service:v2.14.0` was built from a PR that added a new Redis cache integration and changed the database connection pool settings.

## Interviewer Question

"A critical production pod is in CrashLoopBackOff state. It keeps restarting every 10 seconds. The application is a Java Spring Boot microservice. Users are getting errors. How do you systematically diagnose and fix the issue?"

## What I Should Think About

- CrashLoopBackOff means the container started, ran briefly, then exited with a non-zero exit code, and Kubernetes is backing off before restarting
- Need to check logs immediately (current and previous)
- Exit code tells you WHY it crashed (1 = app error, 137 = OOM killed, 139 = segfault, 143 = SIGTERM)
- Common causes: OOM, missing ConfigMap/Secret, bad health check (liveness probe), DB connection failure, bad configuration, application bug
- Must consider the deployment context: was this a new deployment or did it break spontaneously?
- Need to check if this is affecting all replicas or just some
- Should verify resource limits, probe configurations, and environment variables
- The Java-specific angle: JVM heap settings, GC overhead, classpath issues

## Ideal Answer

"Start by gathering information systematically. First, get the pod status and events to understand the failure pattern:

```bash
kubectl get pods -n production -l app=order-service
kubectl describe pod <crashing-pod> -n production
```

Then check the logs — both current and previous crashes:

```bash
kubectl logs <pod-name> -n production --tail=100
kubectl logs <pod-name> -n production --previous --tail=100
```

Check the exit code from the pod status. An exit code of 137 means OOM killed, 1 means application error, 2 means misuse of shell command, 139 means segfault, 143 means graceful SIGTERM.

For a Java application specifically, check:
- JVM heap settings vs container memory limits
- Application logs for Spring Boot startup failures
- Database connectivity
- ConfigMap and Secret availability
- Liveness probe configuration (is it killing the pod before it finishes starting?)

Then correlate the timeline: the crash started 30 minutes ago after a deployment. Check the deployment diff, the new Redis configuration, and the database connection pool changes. If the exit code is 137, the JVM is likely exceeding the container memory limit. If it's 1, the application is throwing an exception during startup."

## Architecture

```
                    ┌─────────────────────────────────┐
                    │         Kubernetes Node          │
                    │                                 │
                    │  ┌───────────────────────────┐  │
                    │  │     Pod: order-service     │  │
                    │  │                           │  │
                    │  │  ┌─────────────────────┐  │  │
                    │  │  │   order-service      │  │  │
                    │  │  │   container          │  │  │
                    │  │  │   (Java/Spring Boot) │  │  │
                    │  │  │                      │  │  │
                    │  │  │  Liveness Probe ─────┼──┼──┼─── Fails → Restart
                    │  │  │  Readiness Probe ────┼──┼──┼─── Fails → No Traffic
                    │  │  │  Resources:          │  │  │
                    │  │  │   requests: 1Gi mem  │  │  │
                    │  │  │   limits:   2Gi mem  │  │  │
                    │  │  └─────────────────────┘  │  │
                    │  └───────────────────────────┘  │
                    │                                 │
                    │  kubelet monitors health checks  │
                    │  and restarts failed containers  │
                    └─────────────────────────────────┘

CrashLoopBackOff Timeline:
  T+0s    Container starts → Spring Boot initializing
  T+8s    Liveness probe fires → App not ready yet → FAIL
  T+16s   Liveness probe fires again → FAIL → kubelet kills container
  T+16s   Restart count: 1 → backoff: 10s
  T+26s   Container starts again
  T+34s   Liveness probe fails again → killed
  T+34s   Restart count: 2 → backoff: 20s
  ...exponential backoff up to 5 minutes
```

## Investigation

**Step 1: Get pod status and identify the failing pods**

```bash
kubectl get pods -n production -l app=order-service -o wide
```

Expected output:
```
NAME                          READY   STATUS             RESTARTS      AGE
order-service-7b9f4d6c8-x2k4p   1/1     Running            0             45m
order-service-7b9f4d6c8-m8j2n   0/1     CrashLoopBackOff   12 (2m ago)   45m
order-service-7b9f4d6c8-p9q7r   0/1     CrashLoopBackOff   12 (2m ago)   45m
```

**Step 2: Describe the crashing pod for events and exit codes**

```bash
kubectl describe pod order-service-7b9f4d6c8-m8j2n -n production
```

Look for:
- Exit codes in the "Last State" section
- Events showing probe failures, OOM kills, or container starts
- Resource limits and requests
- Volume mounts
- Environment variables

**Step 3: Check current and previous logs**

```bash
kubectl logs order-service-7b9f4d6c8-m8j2n -n production --tail=200
kubectl logs order-service-7b9f4d6c8-m8j2n -n production --previous --tail=200
```

**Step 4: Check if it's a liveness probe issue**

```bash
kubectl get pod order-service-7b9f4d6c8-m8j2n -n production -o jsonpath='{.spec.containers[*].livenessProbe}'
```

**Step 5: Verify ConfigMaps and Secrets exist**

```bash
kubectl get configmap -n production | grep order
kubectl get secret -n production | grep order
kubectl get configmap order-config -n production -o yaml
```

**Step 6: Check resource usage if pod starts briefly**

```bash
kubectl top pod order-service-7b9f4d6c8-m8j2n -n production
```

**Step 7: Check the deployment history**

```bash
kubectl rollout history deployment/order-service -n production
kubectl rollout undo deployment/order-service -n production --to-revision=3
```

## Commands

```bash
# Quick overview of all pods and their health
kubectl get pods -n production -l app=order-service -o wide

# Detailed pod status with exit codes
kubectl get pod <pod-name> -n production -o jsonpath='{range .status.containerStatuses[*]}{.name}{"\t"}{.state}{"\t"}{.lastState}{"\t"}{.restartCount}{"\n"}{end}'

# Get pod events (most useful for CrashLoopBackOff)
kubectl describe pod <pod-name> -n production | grep -A 20 "Events:"

# Check current logs
kubectl logs <pod-name> -n production --tail=100 -f

# Check logs from previous crashed container
kubectl logs <pod-name> -n production --previous --tail=100

# Check all containers in the pod (sidecars, init containers)
kubectl logs <pod-name> -n production -c <container-name>

# Check init container logs if present
kubectl logs <pod-name> -n production -c init-db-schema

# Get exit code directly
kubectl get pod <pod-name> -n production -o jsonpath='{.status.containerStatuses[*].lastState.terminated.exitCode}'

# Verify resource limits
kubectl get pod <pod-name> -n production -o jsonpath='{.spec.containers[*].resources}'

# Check liveness and readiness probes
kubectl get pod <pod-name> -n production -o jsonpath='{.spec.containers[*].livenessProbe}' | jq .
kubectl get pod <pod-name> -n production -o jsonpath='{.spec.containers[*].readinessProbe}' | jq .

# Quick rollback to previous version
kubectl rollout undo deployment/order-service -n production

# Rollback to specific revision
kubectl rollout history deployment/order-service -n production
kubectl rollout undo deployment/order-service -n production --to-revision=2

# Exec into a running pod to debug (if it stays up briefly)
kubectl exec -it <pod-name> -n production -- /bin/sh

# Check if there's a postStart hook failing
kubectl get pod <pod-name> -n production -o jsonpath='{.spec.containers[*].lifecycle}' | jq .

# Watch pod status in real time
kubectl get pods -n production -l app=order-service -w
```

## Root Cause

| Root Cause | Exit Code | Evidence | How to Eliminate |
|---|---|---|---|
| **OOM Killed** | 137 | `lastState.terminated.reason: OOMKilled`, `dmesg` shows memory kill | Increase memory limit or reduce JVM heap (`-Xmx`) |
| **Application Error** | 1 | Stack trace in logs, Spring Boot startup exception | Fix code bug, check database connectivity, verify config |
| **Liveness Probe Killing Pod** | 137 | Probe fails before app finishes starting | Increase `initialDelaySeconds`, increase `timeoutSeconds` |
| **Missing Secret/ConfigMap** | 1 | `ConfigMap "xxx" not found` in events | Create the missing resource |
| **Database Connection Failure** | 1 | Connection timeout in logs | Verify DB is running, check connection string, check network policies |
| **Bad Image/Tag** | 0 or 1 | Image pull succeeds but app crashes immediately | Verify image tag, check Dockerfile, test locally |
| **Crash in Post-Start Hook** | 1 | Events show postStart hook failed | Fix the hook script |
| **Init Container Failure** | 1 | Init container never completes | Check init container logs and dependencies |

## Immediate Mitigation

```bash
# 1. Rollback to last known good version IMMEDIATELY
kubectl rollout undo deployment/order-service -n production

# 2. If rollback doesn't work, scale up healthy replicas
kubectl scale deployment order-service -n production --replicas=5

# 3. If the issue is a liveness probe killing too aggressively, patch it
kubectl patch deployment order-service -n production --type='json' -p='[
  {"op": "replace", "path": "/spec/template/spec/containers/0/livenessProbe/initialDelaySeconds", "value": 60},
  {"op": "replace", "path": "/spec/template/spec/containers/0/livenessProbe/periodSeconds", "value": 30}
]'

# 4. If OOM is suspected, temporarily increase memory limit
kubectl patch deployment order-service -n production --type='json' -p='[
  {"op": "replace", "path": "/spec/template/spec/containers/0/resources/limits/memory", "value": "4Gi"}
]'

# 5. Notify stakeholders about the rollback and that service is recovering
```

## Permanent Fix

```yaml
# Fix 1: Increase liveness probe delays for slow-starting Java apps
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
      - name: order-service
        image: registry.internal/order-service:v2.14.1
        resources:
          requests:
            memory: "1536Mi"
            cpu: "500m"
          limits:
            memory: "3Gi"
            cpu: "1000m"
        env:
        - name: JAVA_OPTS
          value: "-Xmx1536m -Xms768m -XX:+UseG1GC -XX:MaxGCPauseMillis=200"
        - name: SPRING_PROFILES_ACTIVE
          value: "production"
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 15
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        startupProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 30
        volumeMounts:
        - name: config
          mountPath: /app/config
          readOnly: true
      volumes:
      - name: config
        configMap:
          name: order-service-config
```

## Monitoring

```yaml
# Add Prometheus monitoring for CrashLoopBackOff detection
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: order-service-crashloop
  namespace: monitoring
spec:
  groups:
  - name: crashloop.rules
    rules:
    - alert: PodCrashLooping
      expr: rate(kube_pod_container_status_restarts_total{namespace="production",pod=~"order-service.*"}[15m]) * 60 * 5 > 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Pod {{ $labels.pod }} is crash looping"
        description: "Pod {{ $labels.pod }} has restarted {{ $value }} times in 15 minutes"

    - alert: PodOOMKilled
      expr: kube_pod_container_status_last_terminated_reason{namespace="production",reason="OOMKilled"} > 0
      for: 0m
      labels:
        severity: critical
      annotations:
        summary: "Pod {{ $labels.pod }} was OOMKilled"

    - alert: DeploymentReplicasMismatch
      expr: kube_deployment_spec_replicas{namespace="production",deployment="order-service"} != kube_deployment_status_ready_replicas{namespace="production",deployment="order-service"}
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "Deployment order-service has mismatched replicas"
```

## Security

- **Secret exposure in logs**: Crash logs may contain database passwords or API keys from failed connection attempts. Ensure log aggregation redacts sensitive data.
- **Image vulnerability**: The crashing image might have known CVEs. Scan with `trivy image registry.internal/order-service:v2.14.0`.
- **RBAC**: Ensure the service account has minimal permissions. A crash loop shouldn't be able to escalate privileges.
- **Resource quotas**: Without proper resource quotas, a crash-looping pod can consume excessive node resources and affect other workloads.
- **Security context**: Run containers as non-root, drop all capabilities, set readOnlyRootFilesystem where possible.

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop:
    - ALL
```

## Production Considerations

- **HPA impact**: CrashLoopBackOff pods confuse HPA. The pod metrics oscillate. Consider using `--horizontal-pod-autoscaler-sync-period` and pod disruption budgets.
- **PodDisruptionBudget**: Always have a PDB to prevent all replicas from being evicted during rolling updates.
- **Progress deadline**: Set `spec.progressDeadlineSeconds` so the deployment fails fast instead of hanging forever.
- **Canary deployments**: Use canary or blue-green deployments to catch issues before full rollout.
- **Circuit breaking**: Implement circuit breakers (Istio/Linkerd) so one crashing service doesn't cascade.
- **Budget**: Every restart costs compute. A pod restarting every 10 seconds consumes significant CPU from the kubelet and etcd.

## Senior-Level Answer

"I'd start by checking the pod's exit code and logs — both current and previous. The exit code tells me the category: 137 is OOM, 1 is application error, 143 is SIGTERM. For this scenario, since it's a Java app that worked in staging but fails in production, I'd suspect either a memory issue (JVM heap exceeding container limits), a missing production ConfigMap/Secret, or a database connectivity difference between environments. I'd check `kubectl describe pod` for events, verify all ConfigMaps and Secrets exist, check if the liveness probe is killing the pod before Spring Boot finishes initializing (common with Java apps that take 30-60 seconds to start), and compare resource limits between staging and production. The fix is usually either increasing `initialDelaySeconds` on the liveness probe, adjusting JVM `-Xmx` to fit within container limits, or fixing the missing resource. I'd also implement a startup probe which is purpose-built for slow-starting applications."

## Architect-Level Answer

"At the architecture level, a CrashLoopBackOff in production reveals gaps in the deployment pipeline and observability. First, this should have been caught in staging — the staging environment needs to mirror production resource limits exactly. Second, I'd implement progressive delivery (canary or blue-green) with automated rollback triggers based on crash rates. Third, I'd enforce that all services expose Spring Boot Actuator health endpoints and configure startup, liveness, and readiness probes correctly in the Helm chart template. Fourth, I'd add OPA/Gatekeeper policies that reject deployments without startup probes or with unreasonable resource limits. Finally, I'd ensure the CI pipeline includes container image scanning, smoke tests against a production-mirror environment, and automated rollback via Argo Rollouts or Flux. The root cause here is likely a process gap — if staging had 4GB and production has 2GB, the staging environment wasn't a faithful production mirror."

## Follow-Up Questions

1. "What's the difference between a liveness probe, readiness probe, and startup probe, and when would you use each?"
2. "How does the exponential backoff work in CrashLoopBackOff? What are the exact timing intervals?"
3. "If the application takes 2 minutes to start due to JVM initialization, how would you configure all three probes to handle this?"
4. "How would you implement automatic rollback in a CI/CD pipeline when CrashLoopBackOff is detected?"
5. "What's the difference between exit code 137 (OOMKilled) caused by the kernel vs exit code 137 caused by the liveness probe killing the container?"
