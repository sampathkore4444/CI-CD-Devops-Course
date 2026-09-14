# 49. Deployment Stuck in Progress

## Scenario

A `kubectl rollout status` shows the `user-service` deployment is stuck at 80% for 20 minutes. The deployment has 5 replicas with a RollingUpdate strategy (maxSurge: 1, maxUnavailable: 0). 4 of 5 new pods are Running and Ready. The 5th new pod is stuck in `ContainerCreating` state. The 4 old pods are still serving traffic. The new version (v2.15.0) has critical bug fixes needed by users. You need to investigate why the 5th pod won't start and unstick the deployment without losing availability.

## Interviewer Question

"A kubectl rollout status shows the deployment is stuck at 80% for 20 minutes. The new pods are Running but not becoming Ready. The old pods are still serving traffic but you need the new features. How do you investigate and unstick the deployment?"

## What I Should Think About

- RollingUpdate with maxUnavailable:0 means ALL old pods must stay running until all new pods are Ready
- 80% means 4/5 new pods are Ready, 1 is stuck
- The stuck pod could be: image pull issue, resource constraint, PVC attachment, node scheduling problem, readiness probe failure
- Need to check pod events, describe pod, check node resources
- The old pods can't be removed until the new pod is Ready (that's what maxUnavailable:0 means)
- Can adjust deployment strategy if needed (increase maxUnavailable to allow progress)
- Need to be careful not to lose availability during troubleshooting

## Ideal Answer

"First, identify why the 5th pod is stuck. Check its events and status:

```bash
kubectl get pods -n production -l app=user-service -o wide
kubectl describe pod <stuck-pod> -n production
```

Common reasons for a pod stuck in ContainerCreating:
1. Image pull backoff (wrong tag, registry unreachable)
2. Node resource constraints (CPU/memory pressure)
3. PersistentVolumeClaim pending (storage not available)
4. Node taints/affinity preventing scheduling
5. Container runtime errors

If the pod is scheduled but ContainerCreating is stuck, check:
- Image pull status: `kubectl get events --field-selector involvedObject.name=<pod>`
- Node resources: `kubectl describe node <node>` (check Allocatable vs Allocated)
- If PVC is pending: `kubectl get pvc -n production`

To unstick:
- If image issue: fix the image reference
- If resource issue: increase node resources or reduce pod requests
- If stuck scheduling: check node taints and pod affinity rules
- Emergency: change strategy to allow more unavailability

The key insight is that with maxUnavailable:0, the deployment won't progress until ALL new pods are ready, which is why old pods can't be removed."

## Architecture

```
RollingUpdate Strategy:
  maxSurge: 1 (at most 1 extra pod during update)
  maxUnavailable: 0 (no pods can be unavailable)

  Timeline:
  ┌─────────────────────────────────────────────────┐
  │ T+0m: 5 old pods serving traffic               │
  │ T+1m: 5 old + 1 new pod created (maxSurge: 1)  │
  │ T+2m: New pod ready → 1 old pod deleted         │
  │ T+3m: 5 old + 1 new pod created                 │
  │ T+4m: New pod ready → 1 old pod deleted         │
  │ ...                                             │
  │ T+15m: 4 new + 1 new pod created (stuck)        │
  │ T+20m: Still stuck at 4/5 new pods              │
  │       4 old pods STILL serving traffic          │
  │       5th new pod: ContainerCreating             │
  │                                                 │
  │  PROBLEM: maxUnavailable:0 prevents removing    │
  │  old pods until ALL new pods are ready          │
  └─────────────────────────────────────────────────┘

Pod Status Breakdown:
  ┌───────────────────────────────────────────┐
  │ user-service-7b9f4d6c8-q1r2s  │ Ready    │ ← New (v2.15.0)
  │ user-service-7b9f4d6c8-t3u4v  │ Ready    │ ← New (v2.15.0)
  │ user-service-7b9f4d6c8-w5x6y  │ Ready    │ ← New (v2.15.0)
  │ user-service-7b9f4d6c8-z7a8b  │ Ready    │ ← New (v2.15.0)
  │ user-service-7b9f4d6c8-c9d0e  │ Stuck!   │ ← New (v2.15.0) - ContainerCreating
  │ user-service-6f8e2a1b3-m1n2o  │ Ready    │ ← Old (v2.14.9)
  │ user-service-6f8e2a1b3-p3q4r  │ Ready    │ ← Old (v2.14.9)
  │ user-service-6f8e2a1b3-s5t6u  │ Ready    │ ← Old (v2.14.9)
  │ user-service-6f8e2a1b3-v7w8x  │ Ready    │ ← Old (v2.14.9)
  └───────────────────────────────────────────┘
```

## Investigation

**Step 1: Get pod status and identify the stuck pod**

```bash
kubectl get pods -n production -l app=user-service -o wide
```

**Step 2: Describe the stuck pod**

```bash
kubectl describe pod <stuck-pod-name> -n production
```

Look for:
- Events: image pull, scheduling, volume attachment
- Conditions: PodScheduled, Initialized, ContainersReady
- Container status: Waiting reason (ImagePullBackOff, ContainerCreating, etc.)

**Step 3: Check pod events**

```bash
kubectl get events -n production --field-selector involvedObject.name=<stuck-pod-name> --sort-by='.lastTimestamp'
```

**Step 4: Check node resources**

```bash
kubectl describe node <node-where-pod-is-scheduled> | grep -A 10 "Allocated resources"
```

**Step 5: Check if PVC is pending**

```bash
kubectl get pvc -n production | grep user-service
```

**Step 6: Check deployment strategy**

```bash
kubectl get deployment user-service -n production -o jsonpath='{.spec.strategy}'
```

**Step 7: Check rollout status**

```bash
kubectl rollout status deployment/user-service -n production
```

**Step 8: Check if image exists**

```bash
kubectl describe pod <stuck-pod-name> -n production | grep -i "image\|pull"
```

## Commands

```bash
# Get all pods with their status
kubectl get pods -n production -l app=user-service -o wide

# Describe the stuck pod
kubectl describe pod <stuck-pod> -n production

# Check events for the stuck pod
kubectl get events -n production --field-selector involvedObject.name=<stuck-pod> --sort-by='.lastTimestamp'

# Check deployment status
kubectl rollout status deployment/user-service -n production

# Check deployment strategy
kubectl get deployment user-service -n production -o jsonpath='{.spec.strategy}' | jq .

# Check node resources
kubectl describe node <node-name> | grep -A 15 "Allocated resources"

# Check PVC status
kubectl get pvc -n production | grep user-service

# Check if the image can be pulled
kubectl get pod <stuck-pod> -n production -o jsonpath='{.spec.containers[*].image}'

# Check image pull secrets
kubectl get pod <stuck-pod> -n production -o jsonpath='{.spec.imagePullSecrets}'

# Temporarily increase maxUnavailable to allow progress
kubectl patch deployment user-service -n production --type='json' -p='[
  {"op": "replace", "path": "/spec/strategy/rollingUpdate/maxUnavailable", "value": "1"}
]'

# Force rollout restart
kubectl rollout restart deployment/user-service -n production

# Scale up to ensure enough replicas
kubectl scale deployment user-service -n production --replicas=8

# Check deployment conditions
kubectl get deployment user-service -n production -o jsonpath='{.status.conditions[*]}' | jq .

# Rollback if needed
kubectl rollout undo deployment/user-service -n production
```

## Root Cause

| Root Cause | Evidence | How to Eliminate |
|---|---|---|
| **Image Pull BackOff** | Events show "Failed to pull image", ImagePullBackOff status | Fix image tag, verify registry credentials, check imagePullSecrets |
| **Node Resource Pressure** | Pod stuck in Pending, node shows CPU/memory pressure | Add nodes, reduce pod resource requests |
| **PVC Pending** | PVC stuck in Pending, no PV available | Create PV, check StorageClass, increase storage capacity |
| **Node Taints** | Pod can't be scheduled, node has taints without tolerations | Add tolerations or remove taints |
| **Readiness Probe Failing** | Pod is Running but not Ready, readiness probe failing | Fix health endpoint, increase probe timeout |
| **Container Runtime Error** | Events show runtime errors | Restart container runtime, check node health |
| **ImagePullSecret Missing** | Events show " unauthorized" | Create imagePullSecret and attach to service account |

## Immediate Mitigation

```bash
# Option 1: Increase maxUnavailable to allow old pods to be removed
kubectl patch deployment user-service -n production --type='json' -p='[
  {"op": "replace", "path": "/spec/strategy/rollingUpdate/maxUnavailable", "value": "1"}
]'

# Option 2: Scale up to create additional new pods
kubectl scale deployment user-service -n production --replicas=8

# Option 3: Delete the stuck pod to let the ReplicaSet create a new one
kubectl delete pod <stuck-pod> -n production

# Option 4: If image is wrong, fix the deployment
kubectl set image deployment/user-service user-service=registry.internal/user-service:v2.15.0-fixed -n production

# Option 5: Rollback if the new version has issues
kubectl rollout undo deployment/user-service -n production
```

## Permanent Fix

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  namespace: production
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 1
  progressDeadlineSeconds: 300
  template:
    spec:
      containers:
      - name: user-service
        image: registry.internal/user-service:v2.15.0
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        resources:
          requests:
            cpu: 250m
            memory: 512Mi
          limits:
            cpu: 500m
            memory: 1Gi
      imagePullSecrets:
      - name: registry-credentials
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: user-service
```

## Monitoring

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: deployment-stuck
spec:
  groups:
  - name: deployment.rules
    rules:
    - alert: DeploymentStuck
      expr: kube_deployment_status_condition{condition="Progressing",status="false"} == 1
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "Deployment {{ $labels.deployment }} is stuck"

    - alert: DeploymentReplicasMismatch
      expr: kube_deployment_spec_replicas != kube_deployment_status_ready_replicas
      for: 15m
      labels:
        severity: warning
      annotations:
        summary: "Deployment {{ $labels.deployment }} has mismatched replicas"
```

## Security

- **Image pull secrets**: Ensure imagePullSecrets are properly configured and not expired
- **Image scanning**: Verify the new image passes security scans before deployment
- **RBAC**: Ensure the deployment service account has minimal permissions

## Production Considerations

- **Progress deadline**: Set `progressDeadlineSeconds` so deployments fail fast instead of hanging
- **Canary deployments**: Use Argo Rollouts for canary deployments with automated rollback
- **Resource quotas**: Ensure namespace resource quotas allow the surge pods
- **PodDisruptionBudget**: Always have a PDB to prevent all pods from being evicted

## Senior-Level Answer

"The deployment is stuck because with maxUnavailable:0, the rolling update won't remove old pods until ALL new pods are Ready. The 5th new pod is stuck in ContainerCreating, which blocks the entire rollout. I'd first check the pod events to determine why it's stuck — likely image pull, resource constraints, or PVC issues. If it's an image issue, I'd fix the image reference. If it's resource pressure, I'd either add nodes or reduce resource requests. As an emergency measure, I'd increase maxUnavailable to 1, which allows the rollout to proceed even if one pod isn't ready. The key learning is that maxUnavailable:0 is very strict — any pod failure blocks the entire deployment. For production, I'd use maxUnavailable:1 with maxSurge:2 and set progressDeadlineSeconds to fail fast."

## Architect-Level Answer

"This scenario reveals a deployment strategy that's too conservative for production. At the architecture level, I'd implement: (1) Canary deployments using Argo Rollouts with automated analysis, (2) Progressive delivery with traffic splitting (10% → 50% → 100%), (3) Automated rollback based on error rate or latency metrics, (4) PodDisruptionBudgets to protect availability during rollouts, and (5) Pre-deployment checks (image scanning, smoke tests) in the CI pipeline. The deployment strategy should balance safety (maxUnavailable:0) with progress (maxUnavailable:1). For critical services, I'd use blue-green deployments with automated canary analysis."

## Follow-Up Questions

1. "What's the difference between maxSurge and maxUnavailable in a RollingUpdate strategy?"
2. "How does Kubernetes determine when a new pod is 'Ready' during a rolling update?"
3. "Explain how progressDeadlineSeconds works and what happens when it's exceeded."
4. "How would you implement a canary deployment in Kubernetes without a service mesh?"
5. "What's the impact of PodDisruptionBudget on rolling update behavior?"
