# 55. Kubernetes Cluster Upgrade Strategy

## Scenario

You need to upgrade a production Kubernetes cluster from 1.27 to 1.29. The cluster runs 200+ pods across 10 namespaces with StatefulSets, DaemonSets, and critical production workloads. The cluster is managed by kubeadm on AWS EC2 instances. There are 3 control plane nodes and 10 worker nodes. The workloads include PostgreSQL StatefulSets, Redis Sentinel, Nginx Ingress Controller, Prometheus monitoring stack, and 50+ microservices. The upgrade must be done with zero downtime. You need to plan and execute the upgrade safely.

## Interviewer Question

"You need to upgrade a production Kubernetes cluster from 1.27 to 1.29. The cluster runs 200+ pods across 10 namespaces with StatefulSets, DaemonSets, and critical production workloads. How do you plan and execute the upgrade safely?"

## What I Should Think About

- Kubernetes must be upgraded one minor version at a time (1.27 to 1.28 to 1.29)
- Control plane must be upgraded before worker nodes
- kubeadm clusters require manual upgrade of each component
- API deprecations must be checked before upgrade
- etcd backup before upgrade is critical
- DaemonSets must tolerate upgrades (node drain)
- StatefulSets need special care (ordered operations, PV reattachment)
- PodDisruptionBudgets must be configured for all critical workloads
- Upgrade CI/CD pipeline with automated checks
- Rollback plan if upgrade fails
- Test in staging first

## Ideal Answer

"The upgrade must follow the Kubernetes skew policy: control plane can be at most 2 versions ahead of worker nodes. For 1.27 to 1.29, I'd upgrade in two steps: 1.27 to 1.28, then 1.28 to 1.29.

Pre-upgrade:
1. etcd snapshot backup
2. Check API deprecations with `kubectl deprecations`
3. Upgrade etcd
4. Upgrade control plane nodes one at a time
5. Upgrade worker nodes one at a time with drain/uncordon
6. Verify each component after upgrade
7. Run smoke tests after each step

For each control plane node, drain it, upgrade kubeadm, run kubeadm upgrade apply, upgrade kubelet and kubectl, then uncordon. For worker nodes, use PodDisruptionBudgets and ensure at least N-1 replicas are always available.

After completing 1.27 to 1.28, repeat the entire process for 1.28 to 1.29. Each step must be validated before proceeding to the next."

## Architecture

```
Cluster Upgrade Sequence:

  Phase 1: Pre-upgrade Checks
  +--------------------------------------------------+
  |  1. etcd snapshot backup                          |
  |  2. API deprecation check (kubectl deprecations)  |
  |  3. Verify PDBs for all critical workloads        |
  |  4. Test upgrade in staging                       |
  |  5. Notify stakeholders                           |
  +--------------------------------------------------+
                        |
                        v
  Phase 2: Control Plane Upgrade (1.27 -> 1.28)
  +--------------------------------------------------+
  |  CP-1: Upgrade kube-apiserver, etcd, controller   |
  |    +-- Verify: kubectl get nodes, kubectl get cs  |
  |                                                   |
  |  CP-2: Upgrade kube-apiserver, etcd, controller   |
  |    +-- Verify: kubectl get nodes, kubectl get cs  |
  |                                                   |
  |  CP-3: Upgrade kube-apiserver, etcd, controller   |
  |    +-- Verify: kubectl get nodes, kubectl get cs  |
  +--------------------------------------------------+
                        |
                        v
  Phase 3: Worker Node Upgrade (1.27 -> 1.28)
  +--------------------------------------------------+
  |  For each worker node (one at a time):            |
  |    1. cordon node                                 |
  |    2. drain node (respect PDBs)                    |
  |    3. upgrade kubelet, kubectl                     |
  |    4. uncordon node                               |
  |    5. verify pods are running                      |
  |    6. wait for stabilization                       |
  +--------------------------------------------------+
                        |
                        v
  Phase 4: Repeat for 1.28 -> 1.29
  +--------------------------------------------------+
  |  Same process as above                            |
  +--------------------------------------------------+
                        |
                        v
  Phase 5: Post-upgrade Validation
  +--------------------------------------------------+
  |  1. Run full test suite                           |
  |  2. Verify all services are healthy               |
  |  3. Check monitoring dashboards                    |
  |  4. Validate API deprecations                      |
  |  5. Update documentation                           |
  +--------------------------------------------------+

  StatefulSet Upgrade Consideration:
  +--------------------------------------------------+
  |  PostgreSQL StatefulSet (3 replicas):             |
  |    - Upgraded in reverse ordinal order             |
  |    - Secondary first, then primary                 |
  |    - PV reattachment needed per node               |
  |    - Test replication after each pod upgrade       |
  +--------------------------------------------------+
```

## Investigation

**Step 1: Check current cluster version**

```bash
kubectl version
kubectl get nodes -o wide
```

**Step 2: Check API deprecations**

```bash
kubectl deprecations --log-level warning
```

**Step 3: Verify etcd health**

```bash
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key
```

**Step 4: Check PDBs**

```bash
kubectl get pdb -A
```

**Step 5: Check addon versions**

```bash
kubectl get pods -n kube-system -o wide
kubectl get deployment -n kube-system
```

**Step 6: Backup etcd**

```bash
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot-$(date +%Y%m%d-%H%M%S).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key
```

## Commands

```bash
# Check current cluster version
kubectl version --short
kubectl get nodes -o wide

# Check API deprecations
kubectl deprecations --log-level warning

# Backup etcd
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key

# Verify etcd backup
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot.db --write-table

# Check PDBs
kubectl get pdb -A

# Check cluster health
kubectl get cs
kubectl get nodes

# Drain a worker node
kubectl drain worker-1 --ignore-daemonsets --delete-emptydir-data --force --grace-period=120

# Upgrade kubeadm on control plane
apt-get update && apt-get install -y kubeadm=1.28.x-00

# Apply upgrade on first control plane node
kubeadm upgrade apply v1.28.x

# Upgrade kubelet and kubectl
apt-get install -y kubelet=1.28.x-00 kubectl=1.28.x-00
systemctl daemon-reload && systemctl restart kubelet

# Uncordon node
kubectl uncordon worker-1

# Upgrade kubeadm on worker node
apt-get update && apt-get install -y kubeadm=1.28.x-00

# Upgrade kubelet config on worker
kubeadm upgrade node

# Upgrade kubelet and kubectl on worker
apt-get install -y kubelet=1.28.x-00 kubectl=1.28.x-00
systemctl daemon-reload && systemctl restart kubelet

# Verify upgrade
kubectl get nodes
kubectl get pods -A

# Check component status
kubectl get cs

# Verify etcd health after upgrade
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key

# Check for deprecated APIs
kubectl get --raw /metrics | grep apiserver_requested_deprecated_apis

# Verify CoreDNS is running
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check addon compatibility
kubectl get deployment -n kube-system -o wide
```

## Root Cause

| Root Cause | Evidence | How to Eliminate |
|---|---|---|
| **Skipped Version** | Upgrade fails, version skew policy violated | Always upgrade one minor version at a time |
| **etcd Corruption** | etcd health check fails after upgrade | Take etcd snapshot before upgrade, verify backup |
| **API Deprecation** | Workloads fail after upgrade, API not found | Run kubectl deprecations before upgrade |
| **Addon Incompatibility** | CoreDNS, CNI, CSI pods CrashLoopBackOff | Upgrade addons to compatible versions |
| **Node Drain Timeout** | Node drain hangs, pods stuck in Terminating | Increase grace period, check PDBs |
| **PV Reattachment** | StatefulSet pods stuck in Pending | Detach volumes from old node before upgrade |
| **CNI Plugin Outdated** | Pods can't communicate after upgrade | Upgrade CNI plugin before cluster upgrade |
| **Container Runtime Incompatible** | kubelet fails to start | Upgrade container runtime first |

## Immediate Mitigation

```bash
# 1. If upgrade fails on control plane, check etcd
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key

# 2. If etcd is corrupted, restore from backup
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restore

# 3. If node is stuck in NotReady, check kubelet
ssh <node> && systemctl status kubelet && journalctl -u kubelet --no-pager -n 50

# 4. If API server is down, check static pod manifests
ssh <cp-node> && crictl ps | grep kube-apiserver

# 5. If CNI is broken, check CNI pod logs
kubectl logs -n kube-system -l app=calico-node --tail=50

# 6. If upgrade is partially complete, complete it manually
kubeadm upgrade node

# 7. Emergency rollback: restore etcd and downgrade kubelet
# (only possible within version skew policy)
```

## Permanent Fix

```yaml
# Automated upgrade script with validation
apiVersion: v1
kind: ConfigMap
metadata:
  name: upgrade-script
  namespace: kube-system
data:
  upgrade-cluster.sh: |
    #!/bin/bash
    set -e
    
    # Pre-upgrade checks
    echo "Running pre-upgrade checks..."
    kubectl deprecations --log-level warning
    kubectl get pdb -A
    
    # Backup etcd
    echo "Backing up etcd..."
    ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%Y%m%d).db \
      --endpoints=https://127.0.0.1:2379 \
      --cacert=/etc/kubernetes/pki/etcd/ca.crt \
      --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
      --key=/etc/kubernetes/pki/etcd/healthcheck-client.key
    
    # Upgrade control plane
    for node in cp-1 cp-2 cp-3; do
      echo "Upgrading $node..."
      kubectl drain $node --ignore-daemonsets --delete-emptydir-data --force
      ssh $node "apt-get update && apt-get install -y kubeadm=1.28.x-00"
      ssh $node "kubeadm upgrade node"
      ssh $node "apt-get install -y kubelet=1.28.x-00 kubectl=1.28.x-00"
      ssh $node "systemctl daemon-reload && systemctl restart kubelet"
      kubectl uncordon $node
      sleep 30
      kubectl get nodes
    done
    
    # Upgrade workers
    for node in worker-{1..10}; do
      echo "Upgrading $node..."
      kubectl drain $node --ignore-daemonsets --delete-emptydir-data --force --grace-period=120
      ssh $node "apt-get update && apt-get install -y kubeadm=1.28.x-00"
      ssh $node "kubeadm upgrade node"
      ssh $node "apt-get install -y kubelet=1.28.x-00 kubectl=1.28.x-00"
      ssh $node "systemctl daemon-reload && systemctl restart kubelet"
      kubectl uncordon $node
      sleep 30
      kubectl get pods -A | grep -v Running | grep -v Completed
    done
    
    # Post-upgrade validation
    echo "Running post-upgrade validation..."
    kubectl get nodes
    kubectl get pods -A
    kubectl get cs
```

## Monitoring

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: upgrade-monitoring
spec:
  groups:
  - name: upgrade.rules
    rules:
    - alert: NodeNotReady
      expr: kube_node_status_condition{condition="Ready",status="true"} == 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Node {{ $labels.node }} is NotReady after upgrade"

    - alert: APIDeprecationWarning
      expr: apiserver_requested_deprecated_apis > 0
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Deprecated API in use: {{ $labels.resource }}"

    - alert: EtcdHighLatency
      expr: etcd_disk_wal_fsync_duration_seconds > 0.5
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "etcd WAL fsync latency high"
```

## Security

- **etcd encryption**: Ensure etcd encryption at rest is enabled before upgrade
- **RBAC changes**: New versions may change RBAC defaults — verify permissions
- **CNI security**: Upgrade CNI plugins to get latest security patches
- **Image scanning**: Scan new kubelet/apiserver images for vulnerabilities
- **Audit logs**: Enable audit logging to track API changes during upgrade

## Production Considerations

- **Maintenance window**: Schedule upgrade during lowest traffic period
- **Communication**: Notify stakeholders before and after upgrade
- **Rollback plan**: Document exact rollback steps for each phase
- **Monitoring**: Watch dashboards throughout upgrade process
- **Automated testing**: Run smoke tests after each phase
- **Canary approach**: Upgrade one node, validate, then proceed
- **Version skew policy**: Control plane max 2 versions ahead of workers

## Senior-Level Answer

"Kubernetes upgrades must follow the version skew policy — one minor version at a time. For 1.27 to 1.29, that means two separate upgrades. The process is: (1) etcd snapshot backup, (2) check API deprecations, (3) upgrade control plane nodes one at a time (drain, upgrade kubeadm, kubeadm upgrade apply, upgrade kubelet, uncordon), (4) upgrade worker nodes one at a time (drain, upgrade kubelet, uncordon), (5) validate after each node. Critical considerations: PodDisruptionBudgets must be configured to maintain availability during drain, StatefulSets need ordered upgrade with PV reattachment verification, and DaemonSets must tolerate the upgrade. The entire process should be tested in staging first, and a rollback plan documented. For managed Kubernetes (EKS, GKE, AKS), the process is simpler — the cloud provider manages control plane upgrades, and you only upgrade node groups."

## Architect-Level Answer

"At the architecture level, cluster upgrades should be: (1) Automated via CI/CD pipeline with validation gates, (2) Tested in staging with production-equivalent workloads, (3) Phased (canary node first, then batch), (4) Monitored with automated rollback triggers, (5) Documented with runbooks. The key architectural decisions are: kubeadm vs managed Kubernetes (EKS/GKE/AKS), upgrade frequency (stay current vs LTS), and automation level (manual vs GitOps). For production, I recommend: (1) Managed Kubernetes where possible (reduces upgrade burden), (2) Automated upgrade pipeline with pre/post validation, (3) PodDisruptionBudgets for all critical workloads, (4) etcd backup automation, and (5) Regular upgrade cadence (quarterly) to avoid large version jumps. The biggest risk is not upgrading — staying on old versions accumulates security vulnerabilities and technical debt."

## Follow-Up Questions

1. "What is the Kubernetes version skew policy and why does it exist?"
2. "How does a kubeadm upgrade differ from a managed Kubernetes (EKS/GKE) upgrade?"
3. "What happens to running pods when a node is drained during an upgrade?"
4. "How do you handle CNI plugin upgrades during a Kubernetes cluster upgrade?"
5. "Explain the difference between in-place upgrade and blue-green cluster upgrade strategies."
