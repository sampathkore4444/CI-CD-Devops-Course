# 58. Pod Scheduling Failure - No Nodes Available

## Scenario

You're deploying a new payment processing microservice to production. The deployment manifest has been reviewed and approved. When you apply it, pods get stuck in `Pending` state. `kubectl describe pod` shows "0/5 nodes are available: 2 had taints that pod didn't tolerate, 3 had insufficient cpu". The cluster has 5 nodes — 2 control plane nodes and 3 worker nodes. The new service requires high CPU (4 cores) and memory (8Gi) for PCI compliance encryption workloads. How do you get these pods scheduled without disrupting existing workloads?

## Interviewer Question

"Pods are stuck in Pending with scheduling failures due to taints and resource constraints. Walk me through diagnosing why pods can't be scheduled and how you'd fix it while maintaining cluster stability."

## What I Should Think About

- Pod scheduling involves multiple constraints: resources, taints/tolerations, affinity/anti-affinity, node selectors
- "No nodes available" is a generic error — need to check `kubectl describe pod` for specifics
- Taints on control plane nodes prevent regular workloads (by design)
- Resource constraints mean nodes are genuinely full
- Check if there are other pods that can be rescheduled or if we need to scale
- Consider if the resource requests are realistic — maybe they're over-provisioned
- DaemonSets, system pods, and kube-system workloads consume resources too
- Node affinity and pod anti-affinity rules can further restrict scheduling

## Ideal Answer

**Step 1: Understand the scheduling failure**

```bash
# Check pod status and events
kubectl get pods -n payment-service
kubectl describe pod <pod-name> -n payment-service | grep -A 20 "Events"

# Check node status
kubectl get nodes -o wide
kubectl describe node worker-1 | grep -A 10 "Allocated resources"
kubectl describe node worker-2 | grep -A 10 "Allocated resources"
kubectl describe node worker-3 | grep -A 10 "Allocated resources"
```

**Step 2: Check taints and tolerations**

```bash
# List all taints on nodes
kubectl get nodes -o json | jq '.items[] | {name: .metadata.name, taints: .spec.taints}'

# Expected output:
# worker-1: no taints
# worker-2: no taints
# worker-3: no taints
# control-plane-1: {"key":"node-role.kubernetes.io/control-plane","effect":"NoSchedule"}
# control-plane-2: {"key":"node-role.kubernetes.io/control-plane","effect":"NoSchedule"}
```

**Step 3: Check resource availability**

```bash
# Check total vs allocated resources per node
kubectl describe nodes | grep -A 5 "Allocated resources"

# Check what's using resources
kubectl top nodes
kubectl top pods --all-namespaces --sort-by=cpu | head -20
```

**Step 4: Fix the issue**

Based on findings, apply one or more solutions:

1. **Scale up nodes** (if cluster is legitimately full):
   ```bash
   # If using cluster autoscaler, check if it's enabled
   kubectl get configmap cluster-autoscaler-status -n kube-system

   # Manually add nodes if autoscaler isn't configured
   ```

2. **Adjust resource requests** (if over-provisioned):
   ```yaml
   resources:
     requests:
       cpu: "2"      # Reduced from 4
       memory: "4Gi"  # Reduced from 8Gi
     limits:
       cpu: "4"
       memory: "8Gi"
   ```

3. **Add toleration for control plane** (if you must run on control plane — NOT recommended):
   ```yaml
   tolerations:
   - key: "node-role.kubernetes.io/control-plane"
     operator: "Equal"
     value: ""
     effect: "NoSchedule"
   ```

4. **Use nodeSelector to target specific nodes**:
   ```yaml
   nodeSelector:
     workload-type: "compute-intensive"
   ```

5. **Use pod anti-affinity to spread across nodes**:
   ```yaml
   affinity:
     podAntiAffinity:
       preferredDuringSchedulingIgnoredDuringExecution:
       - weight: 100
         podAffinityTerm:
           labelSelector:
             matchExpressions:
             - key: app
               operator: In
               values:
               - payment-service
           topologyKey: kubernetes.io/hostname
   ```

## Architecture

```
    Cluster Scheduling Decision Tree
    ─────────────────────────────────

    ┌─────────────────┐
    │  New Pod Created │
    └────────┬────────┘
             │
             ▼
    ┌─────────────────┐     No     ┌──────────────────┐
    │ Any nodes with   │───────────▶│ Pending: No nodes │
    │ matching labels? │            │ available          │
    └────────┬────────┘            └──────────────────┘
             │ Yes
             ▼
    ┌─────────────────┐     No     ┌──────────────────┐
    │ Nodes with no    │───────────▶│ Pending: Taints   │
    │ NoExecute taints?│            │ not tolerated      │
    └────────┬────────┘            └──────────────────┘
             │ Yes
             ▼
    ┌─────────────────┐     No     ┌──────────────────┐
    │ Sufficient       │───────────▶│ Pending: Insufficient │
    │ CPU/Memory?      │            │ resources           │
    └────────┬────────┘            └──────────────────┘
             │ Yes
             ▼
    ┌─────────────────┐     No     ┌──────────────────┐
    │ Affinity rules   │───────────▶│ Pending: Affinity │
    │ satisfied?       │            │ constraints        │
    └────────┬────────┘            └──────────────────┘
             │ Yes
             ▼
    ┌─────────────────┐
    │ Pod Scheduled!  │
    └─────────────────┘

    Current State:
    ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
    │ worker-1         │ │ worker-2         │ │ worker-3         │
    │ CPU: 6/8 cores   │ │ CPU: 7/8 cores   │ │ CPU: 5/8 cores   │
    │ Mem: 14/16Gi     │ │ Mem: 15/16Gi     │ │ Mem: 12/16Gi     │
    │ Remaining: 2 CPU │ │ Remaining: 1 CPU │ │ Remaining: 3 CPU │
    │                  │ │                  │ │ (but needs 4 CPU)│
    └──────────────────┘ └──────────────────┘ └──────────────────┘

    → No single node can fit 4 CPU request
    → Need to either add nodes or reduce requests
```

## Investigation

1. **Check pod describe output** — the Events section tells you exactly why scheduling failed
2. **Check node taints** — control plane nodes have `NoSchedule` taints by default
3. **Check node resource allocation** — are nodes genuinely full?
4. **Check resource requests vs limits** — are requests too high?
5. **Check node selectors and affinity rules** — are they too restrictive?
6. **Check if cluster autoscaler is enabled and functioning**
7. **Check kube-system pods** — DaemonSets consume resources on every node
8. **Check for Evicted pods** — they still consume resources
9. **Check PodDisruptionBudgets** — they might prevent rescheduling
10. **Check if nodes are cordoned** — `kubectl cordon` prevents scheduling

## Commands

```bash
# Comprehensive node analysis
kubectl get nodes -o custom-columns=\
'NAME:.metadata.name,CPU_CAPACITY:.status.capacity.cpu,CPU_ALLOCATED:.status.allocatable.cpu,MEM_CAPACITY:.status.capacity.memory,MEM_ALLOCATED:.status.allocatable.memory,TAINTS:.spec.taints'

# Check which pods are using the most resources
kubectl top pods --all-namespaces --sort-by=cpu

# Check pending pods across all namespaces
kubectl get pods --all-namespaces --field-selector=status.phase=Pending

# Check node conditions
kubectl get nodes -o json | jq '.items[] | {name: .metadata.name, conditions: [.status.conditions[] | select(.type != "Ready")] | map({type: .type, status: .status, reason: .reason})}'

# Check if cluster autoscaler is enabled
kubectl get deployment cluster-autoscaler -n kube-system -o yaml 2>/dev/null

# Check node labels
kubectl get nodes --show-labels

# Check which pods have been evicted
kubectl get pods --all-namespaces --field-selector=status.phase=Failed | grep Evicted
```

## Root Cause

| Root Cause | Evidence | Fix |
|---|---|---|
| Control plane taints | "2 had taints that pod didn't tolerate" | Don't run workloads on control plane (best practice) |
| Insufficient CPU | "3 had insufficient cpu" | Reduce requests, scale up nodes, or add new nodes |
| Insufficient Memory | "X had insufficient memory" | Reduce requests or add nodes |
| Node selector mismatch | Pod has nodeSelector that no node matches | Update labels or nodeSelector |
| Pod anti-affinity | Anti-affinity rules prevent co-location | Relax anti-affinity or add more nodes |
| Nodes cordoned | `kubectl cordon` on nodes | Uncordon if not needed: `kubectl uncordon <node>` |
| No matching node labels | Workload-type labels don't exist | Add labels to nodes |

## Immediate Mitigation

```bash
# Option 1: Add new worker node (if using managed Kubernetes)
# EKS:
aws eks update-nodegroup-config \
  --cluster-name production \
  --nodegroup-name workers \
  --scaling-config minSize=4,maxSize=10,desiredSize=5

# Option 2: Reduce resource requests temporarily
# Edit deployment
kubectl edit deployment payment-service -n payment-service
# Change requests.cpu from "4" to "2", requests.memory from "8Gi" to "4Gi"

# Option 3: Cordon full nodes and let scheduler try others
kubectl cordon worker-2  # Most loaded node

# Option 4: Drain a node to make room
kubectl drain worker-2 --ignore-daemonsets --delete-emptydir-data

# Option 5: Add labels to enable scheduling on existing nodes
kubectl label nodes worker-3 workload-type=compute-intensive
```

## Permanent Fix

1. **Right-size resource requests** — use Vertical Pod Autoscaler (VPA) recommendations:
   ```bash
   kubectl get vpa -n payment-service
   ```

2. **Enable Cluster Autoscaler** — auto-add nodes when scheduling fails:
   ```yaml
   # cluster-autoscaler configmap
   scale-down-utilization-threshold: "0.5"
   max-node-provision-time: "5m"
   ```

3. **Implement Pod Disruption Budgets** — prevent too many pods from being evicted simultaneously

4. **Node pool strategy** — separate node pools for different workload types:
   - General purpose (web, API)
   - Compute intensive (payment processing)
   - Memory intensive (caching, databases)

5. **Resource quotas per namespace** — prevent unbounded resource consumption

## Monitoring

```yaml
- alert: PodsPending
  expr: kube_pod_status_phase{phase="Pending"} > 0
  for: 10m
  labels:
    severity: warning

- alert: NodesNotReady
  expr: kube_node_status_condition{condition="Ready",status="true"} == 0
  for: 5m
  labels:
    severity: critical

- alert: NodeResourcePressure
  expr: kube_node_status_condition{condition="MemoryPressure",status="true"} == 1
  for: 5m
  labels:
    severity: warning
```

Monitor scheduling metrics:
```bash
# Scheduling duration
kube_scheduler_scheduling_algorithm_duration_seconds
kube_scheduler_e2e_scheduling_duration_seconds

# Pending pods
kube_pod_status_phase{phase="Pending"}
```

## Security

- Don't schedule production workloads on control plane nodes (security risk)
- Use node taints to isolate workload types
- Implement ResourceQuotas and LimitRanges per namespace
- Use PodSecurityPolicy or PodSecurityAdmission to restrict pod capabilities
- Monitor for unauthorized scheduling to privileged nodes

## Production Considerations

- **HA**: Spread pods across nodes and AZs using anti-affinity
- **Scalability**: Use Cluster Autoscaler for dynamic node scaling
- **Reliability**: Set resource requests accurately — too high wastes resources, too low causes OOMKills
- **Cost**: Right-sizing nodes and pods reduces waste
- **Operational**: Use node pools for different workload types
- **Compliance**: PCI workloads on dedicated nodes with isolation

## Senior-Level Answer

"Scheduling failures indicate the cluster can't fit new pods given current constraints. First, I check `kubectl describe pod` to understand the specific failure — is it taints, resources, or affinity? For taints, control plane nodes should never run workloads by design. For resource constraints, I check if nodes are genuinely full and whether resource requests are over-provisioned. I'd reduce requests if VPA recommends it, add nodes if needed, or ensure cluster autoscaler is properly configured. Long-term, I'd implement node pools for different workload types and use PodDisruptionBudgets to maintain availability during scaling events."

## Architect-Level Answer

"Pod scheduling failures are a symptom of insufficient capacity planning or overly restrictive constraints. The architectural approach is: (1) right-size workloads using VPA recommendations, (2) implement Cluster Autoscaler with proper scaling policies, (3) design node pools for different workload characteristics (compute, memory, storage), (4) use PodDisruptionBudgets and topology spread constraints for HA, (5) implement ResourceQuotas per namespace to prevent resource hoarding, (6) regularly review and clean up unused resources. The key principle is to separate capacity planning from scheduling — ensure the cluster has enough headroom, then let the scheduler make optimal placement decisions."

## Follow-Up Questions

1. "How does the Kubernetes scheduler make placement decisions when multiple nodes have sufficient resources?"
2. "What's the difference between PodDisruptionBudget and PodDisruptionBudget with maxUnavailable?"
3. "How would you implement a blue-green deployment strategy that requires zero downtime during node maintenance?"
4. "Explain how topology spread constraints differ from pod anti-affinity and when you'd use each."
5. "If the cluster autoscaler isn't adding nodes despite pending pods, what would you check?"
