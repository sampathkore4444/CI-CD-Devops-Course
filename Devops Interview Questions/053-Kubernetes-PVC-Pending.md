# 53. PersistentVolumeClaim Stuck in Pending State

## Scenario

A StatefulSet `postgres-db` in the `database` namespace requires PersistentVolumeClaims. The StatefulSet has 3 replicas, but only `postgres-db-0` is running. PVCs `data-postgres-db-1` and `data-postgres-db-2` are stuck in Pending state. The database cannot form a proper cluster without all replicas. The storage class `fast-ssd` is configured with a provisioner that uses AWS EBS. The cluster has available EBS volumes in the region. The PVC events show "waiting for a volume to be created." You need to diagnose whether it's a storage class issue, capacity issue, access mode issue, or node affinity issue.

## Interviewer Question

"A StatefulSet requires PersistentVolumeClaims. The PVCs are stuck in Pending state. The application (a database) cannot start without storage. How do you diagnose whether it's a storage class issue, capacity issue, or access mode issue?"

## What I Should Think About

- PVC Pending means the dynamic provisioner hasn't created a PV yet
- Common causes: storage class misconfigured, provisioner error, node affinity mismatch, zone/region mismatch, IOPS limits exceeded, IAM permissions issue
- Check PVC events for the exact error message
- Verify the storage class exists and is correctly configured
- Check if the provisioner has the right IAM permissions to create volumes
- Check if there are zone constraints (nodes in one zone, storage in another)
- Verify volume limits per node (AWS has limits on EBS volumes per instance)
- Check if the PVC size is within allowed limits
- Verify the access mode is supported by the storage class

## Ideal Answer

"Check the PVC events first — they tell you exactly why provisioning is stuck:

```bash
kubectl describe pvc data-postgres-db-1 -n database
```

Common error patterns:
- `waiting for a volume to be created` — provisioner hasn't created the PV yet
- `storageclass "fast-ssd" not found` — wrong storage class name
- `exceeded volume limit` — node has too many EBS volumes attached
- `no persistent volumes available` — static PV provisioning issue
- `zone mismatch` — node and PV in different availability zones

Then check:
1. Storage class exists and is correct
2. Provisioner pod is running and healthy
3. Node has capacity for more volumes
4. IAM permissions for the provisioner
5. Zone/region constraints

The most common issue with EBS is node affinity — the PV is created in one zone, but the pod is scheduled in another zone where the volume can't be attached."

## Architecture

```
PVC Provisioning Flow:

  PVC Created → StorageClass → Provisioner → Cloud API → PV Created → PVC Bound
                                              │
                                              ├── IAM check
                                              ├── Zone check
                                              ├── Volume limit check
                                              └── IOPS/throughput check

  Common Failure Points:
  ┌──────────────────────────────────────────────────────┐
  │ 1. StorageClass doesn't exist                        │
  │    → Provisioner never invoked                       │
  │                                                      │
  │ 2. Provisioner pod not running                       │
  │    → No process to handle provisioning               │
  │                                                      │
  │ 3. IAM permissions missing                           │
  │    → AWS API call fails                              │
  │                                                      │
  │ 4. Node zone ≠ PV zone (EBS)                         │
  │    → Volume can't attach to node                     │
  │                                                      │
  │ 5. Volume limit exceeded                             │
  │    → Instance type allows max 39 EBS volumes         │
  │                                                      │
  │ 6. Insufficient IOPS/throughput                      │
  │    → EBS gp3/io2 limits exceeded                     │
  └──────────────────────────────────────────────────────┘

  Node Affinity Problem:
  ┌──────────────────────────────────────────────────────┐
  │  Node-1 (us-east-1a)                                │
  │    └── Pod postgres-db-0 scheduled here              │
  │    └── EBS volume created in us-east-1a ✓            │
  │                                                      │
  │  Node-2 (us-east-1b)                                │
  │    └── Pod postgres-db-1 scheduled here              │
  │    └── EBS volume created in us-east-1a ✗             │
  │    └── CAN'T ATTACH — wrong zone!                    │
  └──────────────────────────────────────────────────────┘
```

## Investigation

**Step 1: Check PVC status and events**

```bash
kubectl get pvc -n database
kubectl describe pvc data-postgres-db-1 -n database
```

**Step 2: Check StorageClass**

```bash
kubectl get storageclass
kubectl describe storageclass fast-ssd
```

**Step 3: Check provisioner pod**

```bash
kubectl get pods -n kube-system | grep -i ebs
kubectl logs -n kube-system -l app=ebs-csi-controller --tail=50
```

**Step 4: Check node volume limits**

```bash
kubectl get node <node-name> -o jsonpath='{.status.capacity}'
kubectl describe node <node-name> | grep -A 5 "Allocated resources"
```

**Step 5: Check IAM permissions (AWS)**

```bash
# Check if the CSI driver service account has the right IAM role
kubectl get sa ebs-csi-controller-sa -n kube-system -o jsonpath='{.metadata.annotations}'
```

**Step 6: Check zone/region constraints**

```bash
kubectl get node <node-name> -o jsonpath='{.metadata.labels.topology\.kubernetes\.io/zone}'
kubectl get pv <pv-name> -o jsonpath='{.spec.nodeAffinity}'
```

**Step 7: Check for volume detach issues**

```bash
kubectl get events -n database --field-selector reason=Provisioning --sort-by='.lastTimestamp'
```

## Commands

```bash
# Check PVC status
kubectl get pvc -n database -o wide

# Describe PVC for events
kubectl describe pvc data-postgres-db-1 -n database

# Check StorageClass
kubectl get sc fast-ssd -o yaml

# Check CSI controller logs
kubectl logs -n kube-system -l app=ebs-csi-controller --tail=100

# Check CSI driver pods
kubectl get pods -n kube-system -l app=ebs-csi-controller
kubectl get pods -n kube-system -l app=ebs-csi-node

# Check node volume limits
kubectl get nodes -o json | jq '.items[] | {name: .metadata.name, volumes: .status.capacity["ephemeral-storage"]}'

# Check PV status
kubectl get pv

# Check if PV exists but isn't bound
kubectl get pv | grep postgres

# Check node topology labels
kubectl get nodes --show-labels | grep topology

# Check provisioner service account IAM role
kubectl get sa -n kube-system ebs-csi-controller-sa -o json

# Manually create PV (emergency)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: manual-pv-1
spec:
  capacity:
    storage: 50Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: fast-ssd
  awsElasticBlockStore:
    volumeID: vol-1234567890abcdef0
    fsType: ext4
EOF

# Force bind PVC to PV
kubectl patch pvc data-postgres-db-1 -n database -p '{"spec":{"volumeName":"manual-pv-1","accessModes":["ReadWriteOnce"]}}'
```

## Root Cause

| Root Cause | Evidence | How to Eliminate |
|---|---|---|
| **StorageClass Not Found** | Events: "storageclass not found" | Create correct StorageClass |
| **Provisioner Not Running** | CSI controller pods not running | Restart CSI controller, check logs |
| **IAM Permissions Missing** | AWS API errors in CSI logs | Attach correct IAM policy to service account |
| **Zone Mismatch** | PVC pending, node in different zone | Use zone-aware scheduling, topology constraints |
| **Volume Limit Exceeded** | Events: "volume limit exceeded" | Use larger instance types, reduce volume count |
| **PVC Size Too Large** | Events: "requested size exceeds limit" | Reduce PVC size or use larger volume type |
| **Node Affinity Conflict** | PV in zone A, pod scheduled in zone B | Use topology-aware provisioning |
| **EBS Volume Stuck Detaching** | Previous volume still attached | Force detach via AWS CLI |

## Immediate Mitigation

```bash
# 1. Check CSI controller logs for exact error
kubectl logs -n kube-system -l app=ebs-csi-controller --tail=100

# 2. If IAM issue, update service account annotation
kubectl annotate sa ebs-csi-controller-sa -n kube-system \
  eks.amazonaws.com/role-arn=arn:aws:iam::ACCOUNT:role/ebs-csi-role

# 3. If zone issue, add topology constraint to StatefulSet
kubectl patch statefulset postgres-db -n database --type='json' -p='[
  {"op": "add", "path": "/spec/template/spec/topologySpreadConstraints/-", "value": {"maxSkew": 1, "topologyKey": "topology.kubernetes.io/zone", "whenUnsatisfiable": "DoNotSchedule", "labelSelector": {"matchLabels": {"app": "postgres-db"}}}}
]'

# 4. Manually create PV from existing EBS volume (emergency)
kubectl get pv manual-pv-1 -o yaml  # Check if PV exists

# 5. Scale down StatefulSet to reduce PVC demand
kubectl scale statefulset postgres-db -n database --replicas=1
```

## Permanent Fix

```yaml
# StorageClass with WaitForFirstConsumer binding mode
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer  # Critical for zone awareness

---
# StatefulSet with topology constraints
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-db
  namespace: database
spec:
  replicas: 3
  template:
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: postgres-db
      containers:
      - name: postgres
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 50Gi
```

## Monitoring

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: pvc-pending
spec:
  groups:
  - name: pvc.rules
    rules:
    - alert: PVCPending
      expr: kube_persistentvolumeclaim_status_phase{phase="Pending"} > 0
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "PVC {{ $labels.persistentvolumeclaim }} is Pending"

    - alert: PVCNotBound
      expr: kube_persistentvolumeclaim_status_phase{phase="Bound"} == 0
      for: 30m
      labels:
        severity: critical
      annotations:
        summary: "PVC {{ $labels.persistentvolumeclaim }} not bound for 30 minutes"
```

## Security

- **Encryption at rest**: Enable EBS encryption in StorageClass parameters
- **IAM least privilege**: CSI driver needs only EBS create/attach/detach/delete permissions
- **Volume deletion protection**: Set reclaimPolicy: Retain for production data
- **Access modes**: Use ReadWriteOncePod instead of ReadWriteOnce for stricter access control

## Production Considerations

- **Volume snapshots**: Configure VolumeSnapshotClass for backup
- **Storage monitoring**: Track PVC usage with Prometheus
- **Capacity planning**: Monitor total storage consumption per node
- **Multi-AZ**: Use topology-aware provisioning for HA databases

## Senior-Level Answer

"PVC Pending means the dynamic provisioner hasn't created a PV. I'd check the PVC events first — they tell you the exact error. Common issues are: storage class not found, IAM permissions for the CSI driver, zone mismatch between node and PV, or volume limit exceeded on the instance. For EBS specifically, the `volumeBindingMode: WaitForFirstConsumer` is critical — it ensures the PV is created in the same zone as the pod. If it's set to `Immediate`, the PV might be created in a different zone. I'd check the CSI controller logs, verify IAM permissions, check node topology labels, and ensure the provisioner pod is running. The fix is usually correcting the StorageClass configuration or fixing IAM permissions."

## Architect-Level Answer

"At the architecture level, storage is often the bottleneck in Kubernetes. I'd implement: (1) `WaitForFirstConsumer` binding mode for all storage classes to ensure zone-aware provisioning, (2) Topology-aware StatefulSets with topology spread constraints, (3) VolumeSnapshotClass for automated backups, (4) Storage monitoring with Prometheus and alerts for PVC usage, (5) Capacity planning per node to prevent volume limit issues, and (6) Consider using a distributed storage solution (Ceph, Portworx) instead of cloud volumes for better HA and portability. The key architectural decision is between cloud-native storage (EBS, persistent disk) and distributed storage — the former is simpler but zone-bound, the latter is more complex but HA."

## Follow-Up Questions

1. "What's the difference between `volumeBindingMode: Immediate` and `WaitForFirstConsumer`?"
2. "How does the EBS CSI driver handle volume attachment and detachment during node failures?"
3. "Explain the difference between PersistentVolume reclaim policies: Retain, Delete, and Recycle."
4. "How would you migrate a StatefulSet's PVCs from one StorageClass to another?"
5. "What are the volume limits for different AWS EC2 instance types?"
