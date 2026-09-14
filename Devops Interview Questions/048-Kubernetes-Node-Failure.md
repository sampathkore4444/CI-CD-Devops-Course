# 48. Kubernetes Node Failure - Critical Workloads Affected

## Scenario

A Kubernetes worker node (node-3) suddenly becomes `NotReady`. The cluster has 5 worker nodes. 15 pods were running on node-3, including: 2 PostgreSQL primary/secondary replicas (StatefulSet), 3 Redis cache instances (StatefulSet), 5 API gateway replicas (Deployment with PDB minAvailable: 2), and 5 worker pods (Deployment). The applications are experiencing partial outages. The API gateway is degraded because only 3 of 5 replicas are running. The database secondary is down, reducing read capacity. You need to understand how Kubernetes handles node failure, ensure workloads recover properly, and handle the special case of StatefulSets with PersistentVolumes.

## Interviewer Question

"A Kubernetes worker node suddenly becomes NotReady. 15 pods were running on it, including 3 database replicas and critical API services. How does Kubernetes handle node failure? How do you ensure workloads recover properly? What about StatefulSets and PersistentVolumes?"

## What I Should Think About

- Node controller marks node as NotReady after `pod-eviction-timeout` (default 5 minutes)
- Taint-based eviction: kubelet applies `node.kubernetes.io/not-ready:NoExecute` taint
- Pods with tolerations might not be evicted
- StatefulSets have ordered lifecycle — secondary can't become primary until secondary pod is rescheduled
- PersistentVolumes are bound to nodes via PersistentVolumeReclaim policies
- PodDisruptionBudgets prevent too many replicas from being evicted simultaneously
- The API server still thinks pods exist until the eviction timeout
- Need to manually cordon and drain if node is permanently failed
- Watch for orphaned PersistentVolumes that need reattachment

## Ideal Answer

"When a node becomes NotReady, the Kubernetes node controller detects it via the kubelet's heartbeat (default every 10 seconds). After the `pod-eviction-timeout` (default 5 minutes), the controller starts evicting pods. Pods are deleted, and their controllers (Deployments, StatefulSets) create replacement pods on healthy nodes.

For Deployments, the ReplicaSet controller immediately creates new pods to maintain the desired replica count. PodDisruptionBudgets limit how many can be evicted simultaneously. For StatefulSets, the replacement is more complex — pods are rescheduled in reverse ordinal order, and PersistentVolumes must be reattached to the new node.

The critical concern is data availability: if the PostgreSQL primary was on the failed node, you need to promote a secondary. If PersistentVolumes are node-local (like EBS volumes), they need to be detached from the failed node and attached to the new node before the pod can start.

Immediate actions: cordon the node to prevent new pods, drain remaining pods if the node won't recover, verify PV reattachment, and monitor pod rescheduling."

## Architecture

```
Node Failure Timeline:

T+0s    Node-3 becomes unreachable (network partition, kernel panic, hardware failure)
        │
T+10s   Kubelet on node-3 stops sending heartbeats
        │
T+30s   Node controller marks node-3 as Unknown
        │
T+5m    pod-eviction-timeout reached
        │
T+5m    Node controller starts evicting pods:
        │   ├── kube-system pods with no toleration → evicted
        │   ├── API gateway pods → evicted (PDB allows 2 evictions)
        │   ├── PostgreSQL secondary → evicted
        │   ├── Redis pods → evicted
        │   └── Worker pods → evicted
        │
T+5m+   Pod controllers react:
        │   ├── ReplicaSet creates new API gateway pods → scheduled to node-1,2,4
        │   ├── StatefulSet creates postgresql-1 pod → scheduled to node-1
        │   ├── StatefulSet creates redis pods → scheduled to healthy nodes
        │   └── Deployment creates worker pods → scheduled to healthy nodes
        │
T+5m+   PersistentVolume reattachment:
        │   ├── EBS volumes: detach from node-3 → attach to new node
        │   ├── NFS/CephFS: already accessible (no detach needed)
        │   └── Local PVs: LOST (data unavailable until node recovers)
        │
T+8m    Pods start on new nodes, databases recover
        │
T+10m   Service endpoints updated, traffic flows to new pods

StatefulSet Recovery:

  PostgreSQL StatefulSet:
  ┌──────────────────────────────────────────────┐
  │  Original:                                   │
  │  postgresql-0 (primary) → node-1 (healthy)   │
  │  postgresql-1 (secondary) → node-3 (FAILED)  │
  │                                              │
  │  After failure:                              │
  │  postgresql-0 (primary) → node-1 (healthy)   │
  │  postgresql-1 (secondary) → node-2 (new)     │
  │    └── Needs PV reattachment                 │
  │    └── Needs replication catch-up             │
  │    └── Must be in Running+Ready before       │
  │        any promotion can occur                │
  └──────────────────────────────────────────────┘

PersistentVolume Flow:
  PV-xxx (EBS vol-12345) → was attached to node-3
       │
       ├── 1. Controller-manager detaches from node-3
       ├── 2. Controller-manager attaches to node-2
       ├── 3. kubelet on node-2 mounts the volume
       └── 4. Pod starts and mounts the volume
```

## Investigation

**Step 1: Confirm node status**

```bash
kubectl get nodes
kubectl describe node node-3
```

**Step 2: Check what pods were affected**

```bash
kubectl get pods --all-namespaces --field-selector spec.nodeName=node-3
```

**Step 3: Check pod eviction status**

```bash
kubectl get pods --all-namespaces --field-selector spec.nodeName=node-3 -o wide
kubectl get events --all-namespaces --field-selector reason=NodeNotReady --sort-by='.lastTimestamp'
```

**Step 4: Verify replacement pods are being created**

```bash
kubectl get pods -n production -o wide
kubectl get statefulset -n production
```

**Step 5: Check PersistentVolume status**

```bash
kubectl get pv
kubectl get pvc -n production
kubectl describe pvc postgres-data-postgresql-1 -n production
```

**Step 6: Monitor pod rescheduling**

```bash
kubectl get pods -n production -o wide --watch
```

**Step 7: Check PDB impact**

```bash
kubectl get pdb -n production
kubectl describe pdb api-gateway-pdb -n production
```

## Commands

```bash
# Node status overview
kubectl get nodes -o wide
kubectl describe node node-3 | grep -A 20 "Conditions:"

# Find all pods that were on the failed node
kubectl get pods --all-namespaces -o json | jq -r '.items[] | select(.spec.nodeName=="node-3") | "\(.metadata.namespace)/\(.metadata.name) \(.status.phase)"'

# Check eviction events
kubectl get events --all-namespaces --sort-by='.lastTimestamp' | grep -i "evict\|kill\|node"

# Cordon the failed node (prevent new pods)
kubectl cordon node-3

# Drain the node (evict remaining pods gracefully)
kubectl drain node-3 --ignore-daemonsets --delete-emptydir-data --force --grace-period=60

# Check StatefulSet status
kubectl get statefulset -n production -o wide
kubectl rollout status statefulset/postgresql -n production

# Check PersistentVolume attachment status
kubectl get pv -o json | jq '.items[] | {name: .metadata.name, phase: .status.phase, claimRef: .status.claimRef, node: .spec.nodeAffinity}'

# Check PVC events for attachment issues
kubectl describe pvc -n production postgres-data-postgresql-1

# Force reattach PV by deleting the pod
kubectl delete pod postgresql-1 -n production

# Uncordon the node when it recovers
kubectl uncordon node-3

# Check node conditions over time
kubectl get node node-3 -o jsonpath='{range .status.conditions[*]}{.type}: {.status} ({.reason}){"\n"}{end}'

# Verify replacement pods are scheduled
kubectl get pods -n production -o wide --field-selector spec.nodeName=node-1

# Check DaemonSet pods (should self-heal)
kubectl get daemonset -n kube-system
```

## Root Cause

| Root Cause | Evidence | How to Eliminate |
|---|---|---|
| **Hardware Failure** | Node unreachable, cloud console shows instance issue | Use multiple node pools, spread across AZs |
| **Kernel Panic** | Node shows kernel panic in serial console logs | Update kernel, monitor node health |
| **Network Partition** | Node reachable but not to API server | Check network, use multiple AZs |
| **Disk Full** | kubelet can't write, node becomes NotReady | Monitor disk usage, use node health checks |
| **Memory Pressure** | OOM killed kubelet, node becomes NotReady | Set proper memory reservations |
| **Cloud Instance Termination** | Spot/preemptible instance terminated | Use node pools with on-demand instances for critical workloads |
| **Kubelet Crash** | kubelet not running, node appears NotReady | Restart kubelet, monitor kubelet health |

## Immediate Mitigation

```bash
# 1. Cordon the failed node immediately
kubectl cordon node-3

# 2. Check if replacement pods are being created
kubectl get pods -n production -o wide --watch

# 3. If StatefulSet pods are stuck, check PV status
kubectl describe pvc postgres-data-postgresql-1 -n production

# 4. If PV is stuck in node-3, manually force detach (cloud-specific)
# AWS example:
aws ec2 detach-volume --volume-id vol-12345 --force

# 5. If API gateway is below PDB, check if enough replicas exist
kubectl get pdb api-gateway-pdb -n production

# 6. If the node is permanently dead, drain it
kubectl drain node-3 --ignore-daemonsets --delete-emptydir-data --force

# 7. Verify database secondary can be promoted if primary was lost
kubectl exec -it postgresql-0 -n production -- psql -c "SELECT pg_is_in_recovery();"
```

## Permanent Fix

```yaml
# 1. PodDisruptionBudget to protect critical workloads
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-gateway-pdb
  namespace: production
spec:
  minAvailable: 3
  selector:
    matchLabels:
      app: api-gateway

---
# 2. Anti-affinity to spread pods across nodes
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
spec:
  replicas: 5
  template:
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: api-gateway
              topologyKey: kubernetes.io/hostname

---
# 3. Topology spread constraints for zone awareness
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgresql
spec:
  template:
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: postgresql

---
# 4. Node affinity for database nodes with more resources
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-type
                operator: In
                values:
                - database
```

## Monitoring

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: node-health
spec:
  groups:
  - name: node.rules
    rules:
    - alert: NodeNotReady
      expr: kube_node_status_condition{condition="Ready",status="true"} == 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Node {{ $labels.node }} is NotReady"

    - alert: NodePodCountHigh
      expr: count by (node) (kube_pod_info) > 50
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Node {{ $labels.node }} has >50 pods"

    - alert: PersistentVolumeDetached
      expr: kube_persistentvolume_status_phase{phase="Released"} > 0
      for: 10m
      labels:
        severity: warning
```

## Security

- **Node compromise**: A NotReady node might be compromised. Investigate before draining.
- **Data at rest**: Ensure PVs are encrypted. If a node is lost, encrypted volumes prevent data exposure.
- **Network isolation**: Use NetworkPolicies to limit what pods can communicate across nodes.

## Production Considerations

- **Multi-AZ**: Spread nodes across availability zones to survive AZ failures
- **Cluster Autoscaler**: After node failure, ensure autoscaler can replace the node
- **Backup strategy**: Regular database backups independent of PV attachment
- **Chaos engineering**: Regularly test node failures with Chaos Monkey or Litmus
- **Monitoring**: Track node health metrics, not just pod health

## Senior-Level Answer

"When a node becomes NotReady, the node controller waits for the pod-eviction-timeout (default 5 minutes), then evicts pods. Deployments immediately reschedule pods on healthy nodes. StatefulSets are more complex — pods are rescheduled with OrderedReady semantics, and PersistentVolumes must be detached from the failed node and reattached. I'd first cordon the node, then verify PV status. If PVs are EBS volumes, they need explicit detachment from the failed instance. For the database, I'd check if the primary was on the failed node and promote a secondary if needed. The key is understanding that Kubernetes handles stateless workloads automatically, but stateful workloads require manual intervention for PV reattachment. I'd also ensure the PDB for the API gateway has enough headroom for a single node failure."

## Architect-Level Answer

"At the architectural level, node failure should be a non-event. I'd implement: (1) Multi-AZ node pools with topology spread constraints, (2) PodDisruptionBudgets for every critical workload, (3) Pod anti-affinity rules to spread replicas across nodes and zones, (4) StatefulSets with volumeClaimTemplates using replicated storage (Ceph, Portworx) instead of node-local EBS, (5) Regular chaos testing to validate resilience, and (6) Cluster autoscaler configured to replace failed nodes automatically. The key architectural decision is choosing between node-local storage (cheaper, faster, but node-coupled) and replicated storage (more expensive, but survives node failures). For production databases, replicated storage is essential."

## Follow-Up Questions

1. "What is the difference between `pod-eviction-timeout` and `taint-based eviction`?"
2. "How does a StatefulSet handle the case where a PersistentVolume is stuck in 'Detaching' state?"
3. "Explain the difference between `kubectl drain` with and without `--force`."
4. "How does the Cluster Autoscaler respond when a node becomes NotReady?"
5. "What happens to DaemonSet pods when a node becomes NotReady?"
