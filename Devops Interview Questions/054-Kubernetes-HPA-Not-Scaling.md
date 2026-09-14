# 54. HPA Not Scaling as Expected

## Scenario

A `HorizontalPodAutoscaler` is configured for the `api-gateway` deployment in the `production` namespace. The HPA target is 70% CPU utilization. Current CPU usage is at 80% (measured by Prometheus), but the HPA shows `current metrics < target` and the replica count remains at 3. The deployment has resource requests set to `cpu: 500m`. The Metrics Server is installed. Users are experiencing slow response times because the pods are overloaded. You need to debug why the HPA isn't scaling despite high CPU usage.

## Interviewer Question

"HorizontalPodAutoscaler is configured but not scaling despite CPU being at 80%. The HPA shows 'current metrics < target.' How do you debug HPA behavior, metrics server issues, and scaling configuration?"

## What I Should Think About

- HPA uses metrics from Metrics Server (CPU/memory) or Prometheus (custom metrics)
- CPU metric in HPA is based on `cpu: request`, not actual CPU usage
- If CPU request is set to 500m but the pod uses 400m, HPA sees 80% but actual might be different
- Metrics Server might have stale or incorrect data
- HPA has stabilization windows that delay scaling
- Min/max replicas might be reached
- The `--horizontal-pod-autoscaler-sync-period` affects how often HPA checks
- Resource requests must be set for CPU-based scaling to work
- Check if HPA is actually targeting the correct deployment

## Ideal Answer

"The key insight is that HPA calculates CPU utilization as: `(current CPU usage / pod CPU request) * 100`. If the pod CPU request is 500m and actual usage is 400m, HPA sees 80%. But if Prometheus shows 80% of node CPU, that's different from pod-level CPU.

First, check what metrics the HPA is actually seeing:

```bash
kubectl get hpa api-gateway -n production
kubectl describe hpa api-gateway -n production
```

The `describe` output shows the exact metrics. If it says `current metrics < target`, the Metrics Server is reporting lower CPU than Prometheus.

Common causes:
1. Metrics Server has stale data (restart it)
2. CPU requests are set too high (inflating the denominator)
3. HPA stabilization window is too long
4. Min/max replicas are at boundaries
5. Metrics Server pod is not running

Fix: verify Metrics Server, check resource requests, adjust HPA configuration."

## Architecture

```
HPA Scaling Decision Flow:

  Metrics Server → HPA Controller → Scale Decision → ReplicaSet Controller → Pods

  CPU Calculation:
  ┌──────────────────────────────────────────────────────┐
  │  HPA CPU Metric = (Actual CPU Usage / CPU Request)   │
  │                                                      │
  │  Example:                                            │
  │    CPU Request: 500m (0.5 cores)                     │
  │    Actual Usage: 400m (0.4 cores)                    │
  │    HPA sees: 400/500 = 80%                           │
  │    Target: 70%                                        │
  │    Current > Target → SHOULD SCALE UP                │
  │                                                      │
  │  But if:                                             │
  │    CPU Request: 1000m (1 core)                       │
  │    Actual Usage: 400m (0.4 cores)                    │
  │    HPA sees: 400/1000 = 40%                          │
  │    Target: 70%                                        │
  │    Current < Target → NO SCALE                        │
  └──────────────────────────────────────────────────────┘

  Metrics Server Data Flow:
  ┌──────────────────────────────────────────────────────┐
  │  1. Kubelet collects metrics from cAdvisor            │
  │  2. Metrics Server aggregates metrics                │
  │  3. HPA controller queries Metrics Server API        │
  │  4. HPA calculates desired replicas                  │
  │  5. HPA patches Deployment spec.replicas             │
  │                                                      │
  │  If step 2 has stale data → HPA makes wrong decision │
  └──────────────────────────────────────────────────────┘
```

## Investigation

**Step 1: Check HPA current state**

```bash
kubectl get hpa api-gateway -n production
kubectl describe hpa api-gateway -n production
```

**Step 2: Check metrics server**

```bash
kubectl get pods -n kube-system | grep metrics-server
kubectl logs -n kube-system -l k8s-app=metrics-server --tail=50
```

**Step 3: Query metrics directly**

```bash
kubectl top pods -n production -l app=api-gateway
kubectl top nodes
```

**Step 4: Check resource requests**

```bash
kubectl get deployment api-gateway -n production -o jsonpath='{.spec.template.spec.containers[*].resources}'
```

**Step 5: Check HPA events**

```bash
kubectl get events -n production --field-selector involvedObject.name=api-gateway --sort-by='.lastTimestamp'
```

**Step 6: Check HPA configuration**

```bash
kubectl get hpa api-gateway -n production -o yaml
```

**Step 7: Check min/max replicas**

```bash
kubectl get hpa api-gateway -n production -o jsonpath='{.spec.minReplicas}'
kubectl get hpa api-gateway -n production -o jsonpath='{.spec.maxReplicas}'
```

## Commands

```bash
# Get HPA status
kubectl get hpa api-gateway -n production

# Describe HPA for detailed metrics
kubectl describe hpa api-gateway -n production

# Check Metrics Server pods
kubectl get pods -n kube-system | grep metrics-server

# Check Metrics Server logs
kubectl logs -n kube-system -l k8s-app=metrics-server --tail=100

# Query pod CPU usage directly
kubectl top pods -n production -l app=api-gateway --containers

# Check resource requests
kubectl get deployment api-gateway -n production -o jsonpath='{.spec.template.spec.containers[*].resources}' | jq .

# Check HPA events
kubectl get events -n production --sort-by='.lastTimestamp' | grep -i hpa

# Test Metrics Server API
kubectl get --raw /apis/metrics.k8s.io/v1beta1/pods | jq .

# Check HPA scaling history
kubectl get hpa api-gateway -n production -o jsonpath='{.status.conditions[*]}' | jq .

# Manually scale (emergency)
kubectl scale deployment api-gateway -n production --replicas=6

# Check if HPA can scale (min/max)
kubectl get hpa api-gateway -n production -o jsonpath='{.spec.minReplicas,.spec.maxReplicas}'

# Check custom metrics (if using Prometheus)
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1" | jq .

# Verify HPA targets correct deployment
kubectl get hpa api-gateway -n production -o jsonpath='{.spec.scaleTargetRef}'
```

## Root Cause

| Root Cause | Evidence | How to Eliminate |
|---|---|---|
| **Metrics Server Stale** | `kubectl top` shows old data | Restart Metrics Server |
| **CPU Request Too High** | HPA sees low % but pods are overloaded | Lower CPU request to match actual usage |
| **HPA Min Replicas at Max** | HPA shows `desiredReplicas=maxReplicas` | Increase maxReplicas |
| **Metrics Server Not Running** | Metrics Server pod CrashLoopBackOff | Fix Metrics Server |
| **Stabilization Window Too Long** | HPA delays scaling | Reduce `behavior.scaleDown.stabilizationWindowSeconds` |
| **Resource Requests Not Set** | HPA can't calculate CPU % | Set CPU requests on containers |
| **Wrong Metric Source** | HPA uses custom metrics that don't exist | Fix metric source or use CPU metrics |
| **Sync Period Too Long** | HPA checks infrequently | Reduce `--horizontal-pod-autoscaler-sync-period` |

## Immediate Mitigation

```bash
# 1. Manually scale to handle load
kubectl scale deployment api-gateway -n production --replicas=6

# 2. Check and fix Metrics Server
kubectl rollout restart deployment metrics-server -n kube-system

# 3. Check resource requests
kubectl get deployment api-gateway -n production -o jsonpath='{.spec.template.spec.containers[0].resources}'

# 4. If CPU request is too high, lower it
kubectl patch deployment api-gateway -n production --type='json' -p='[
  {"op": "replace", "path": "/spec/template/spec/containers/0/resources/requests/cpu", "value": "250m"}
]'

# 5. Increase HPA maxReplicas
kubectl patch hpa api-gateway -n production --type='json' -p='[
  {"op": "replace", "path": "/spec/maxReplicas", "value": 10}
]'
```

## Permanent Fix

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-gateway
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-gateway
  minReplicas: 3
  maxReplicas: 15
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 100
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
  namespace: production
spec:
  template:
    spec:
      containers:
      - name: api-gateway
        resources:
          requests:
            cpu: 250m
            memory: 512Mi
          limits:
            cpu: 1000m
            memory: 1Gi
```

## Monitoring

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: hpa-monitoring
spec:
  groups:
  - name: hpa.rules
    rules:
    - alert: HPANotScaling
      expr: kube_horizontalpodautoscaler_status_current_replicas == kube_horizontalpodautoscaler_spec_max_replicas
        and kube_horizontalpodautoscaler_status_current_replicas == kube_horizontalpodautoscaler_status_desired_replicas
      for: 15m
      labels:
        severity: warning
      annotations:
        summary: "HPA {{ $labels.horizontalpodautoscaler }} at max replicas"

    - alert: HPAUtilizationHigh
      expr: kube_horizontalpodautoscaler_status_current_metrics_utilization > 90
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "HPA {{ $labels.horizontalpodautoscaler }} utilization > 90%"
```

## Security

- **Metrics Server access**: Restrict who can query metrics
- **HPA permissions**: Ensure HPA has RBAC to scale deployments
- **Resource quotas**: Set resource quotas to prevent runaway scaling
- **Cost control**: Set maxReplicas to prevent unexpected cost spikes

## Production Considerations

- **Custom metrics**: Use Prometheus Adapter for custom metrics (request latency, queue depth)
- **KEDA**: Consider KEDA for event-driven autoscaling
- **Cluster Autoscaler**: Ensure cluster autoscaler adds nodes when HPA scales pods
- **Cost monitoring**: Track scaling events and associated costs

## Senior-Level Answer

"The HPA sees CPU as `(actual usage / CPU request) * 100`. If the CPU request is set to 1000m but actual usage is 400m, HPA sees 40% — below the 70% target — so it won't scale. But the pods are actually overloaded. I'd check: (1) `kubectl describe hpa` to see the exact metrics, (2) `kubectl top pods` to see actual CPU usage, (3) verify CPU requests are set correctly. If CPU request is too high, lower it. If Metrics Server is stale, restart it. The fix is usually adjusting CPU requests to match actual usage patterns, or switching to custom metrics (like request latency) that better represent load."

## Architect-Level Answer

"At the architecture level, CPU-based HPA is often a poor proxy for actual load. I'd implement: (1) Custom metrics via Prometheus Adapter (request latency, error rate, queue depth), (2) KEDA for event-driven scaling (Kafka consumers, SQS queue depth), (3) Combined HPA with multiple metrics (CPU + custom), (4) PodDisruptionBudget to protect availability during scaling, and (5) Cluster Autoscaler integration to add nodes when pods can't be scheduled. The key architectural decision is: what metric actually represents 'load' for this service? CPU is often not the answer — request latency, error rate, or queue depth are better indicators."

## Follow-Up Questions

1. "How does HPA calculate desired replicas? Walk through the formula."
2. "What's the difference between `averageUtilization` and `averageValue` in HPA metrics?"
3. "How does the HPA stabilization window affect scaling behavior?"
4. "Explain how KEDA differs from HPA for event-driven workloads."
5. "How does the Cluster Autoscaler interact with HPA when pods can't be scheduled?"
