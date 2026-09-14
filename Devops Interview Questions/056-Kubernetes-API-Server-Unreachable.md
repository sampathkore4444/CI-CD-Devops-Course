# 56. Kubernetes API Server Unreachable

## Scenario

It's 2:47 AM on a Saturday. You get paged because `kubectl` commands are timing out across all environments. The CI/CD pipeline is failing because it can't apply manifests. The on-call dashboard shows all services are healthy and serving traffic normally. Pods are running, ingresses are routing, and monitoring is collecting metrics. But nobody can deploy, scale, or manage anything in the cluster. The CEO wants to know if we need to wake up the entire engineering team or if this is manageable. You need to figure out what's wrong with the API server and restore access without disrupting running workloads.

## Interviewer Question

"The Kubernetes API server is unreachable via kubectl but all pods are running and serving traffic normally. Walk me through your troubleshooting process. How do you diagnose the issue, restore access, and ensure this doesn't happen again?"

## What I Should Think About

- API server being down doesn't mean workloads are down — pods run independently on kubelet
- Check connectivity at multiple layers: network, DNS, TLS, authentication
- Multiple ways to access: kubectl, direct API server URL, bastion hosts, jump boxes
- etcd might be the underlying issue — it's the backing store for API server
- Control plane components run on specific nodes — check those first
- Load balancer in front of API server could be the bottleneck
- Certificate expiration is a common and sneaky cause
- Resource exhaustion on control plane nodes

## Ideal Answer

**First, determine the scope**: Check if the issue is local (your kubeconfig) or cluster-wide (API server itself).

```bash
# Test direct API server connectivity
kubectl cluster-info
curl -k https://kubernetes.default.svc.cluster.local:6443/healthz
curl -k https://<API_SERVER_IP>:6443/healthz

# Check from within the cluster
kubectl run debug --image=busybox --rm -it --restart=Never -- wget -qO- --timeout=5 https://kubernetes.default.svc:6443/healthz
```

**If the API server is down**, check the control plane pods:

```bash
# SSH into a control plane node
ssh ubuntu@control-plane-1

# Check container runtime
sudo crictl ps | grep -E "kube-apiserver|etcd|kube-scheduler|kube-controller"

# Check kubelet
sudo systemctl status kubelet
sudo journalctl -u kubelet --since "30 minutes ago" --no-pager

# Check API server logs
sudo crictl logs $(sudo crictl ps -q --name kube-apiserver) --since 30m

# Check etcd health
sudo crictl logs $(sudo crictl ps -q --name etcd) --since 30m
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
  endpoint health
```

**Common root causes and fixes**:

1. **Certificate expiration**: Check certificate validity:
   ```bash
   openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -dates
   ```
   If expired, regenerate with kubeadm.

2. **etcd failure**: If etcd is down, the API server can't serve requests.
   ```bash
   # Restart etcd
   sudo crictl stop $(sudo crictl ps -q --name etcd)
   # etcd should auto-restart via static pod manifest
   ```

3. **Load balancer failure**: If using an LB in front of API servers:
   ```bash
   # Check ALB/NLB health
   aws elbv2 describe-target-health --target-group-arn <arn>
   ```

4. **Resource exhaustion**: Check control plane node resources:
   ```bash
   top -bn1 | head -20
   df -h /var/lib/etcd
   free -m
   ```

**Restore access** while investigating — use kubectl with `--server` flag pointing to an alternative endpoint or SSH into a control plane node and use local kubeconfig.

## Architecture

```
                    ┌──────────────────────────────────────┐
                    │         User / CI/CD Pipeline         │
                    └──────────────┬───────────────────────┘
                                   │
                                   ▼
                    ┌──────────────────────────────────────┐
                    │     Load Balancer (NLB/ALB)          │
                    │     (k8s API endpoint:6443)          │
                    └────────┬───────────────┬─────────────┘
                             │               │
                    ┌────────▼──────┐ ┌──────▼────────┐
                    │ Control Plane │ │ Control Plane │
                    │   Node 1      │ │   Node 2      │
                    │ ┌───────────┐ │ │ ┌───────────┐ │
                    │ │API Server │ │ │ │API Server │ │
                    │ │Scheduler  │ │ │ │Scheduler  │ │
                    │ │Controller │ │ │ │Controller │ │
                    │ │  Manager  │ │ │ │  Manager  │ │
                    │ ├───────────┤ │ │ ├───────────┤ │
                    │ │   etcd    │ │ │ │   etcd    │ │
                    │ └───────────┘ │ │ └───────────┘ │
                    └───────────────┘ └───────────────┘
                             │               │
                    ┌────────▼───────────────▼─────────────┐
                    │           Worker Nodes               │
                    │  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐   │
                    │  │Pod 1│ │Pod 2│ │Pod 3│ │Pod N│   │
                    │  └─────┘ └─────┘ └─────┘ └─────┘   │
                    └──────────────────────────────────────┘

    Problem: API server unreachable, but worker nodes and pods are fine
    → Workers run independently via kubelet (local control loop)
    → Only management plane is affected
```

## Investigation

1. **Check your local kubeconfig** — is the server URL correct? Is the context right?
2. **Test connectivity** — can you reach the API server IP:6443 from your machine?
3. **Check DNS resolution** — does the API server hostname resolve?
4. **Test TLS** — is the certificate valid and not expired?
5. **Check authentication** — are your credentials (token/cert) valid?
6. **Check control plane node health** — SSH into a control plane node
7. **Check kubelet on control plane** — is it running the static pods?
8. **Check etcd health** — is etcd responding and not out of disk/memory?
9. **Check API server logs** — what errors are being logged?
10. **Check load balancer** — if one exists, is it routing correctly?

## Commands

```bash
# From your workstation
kubectl config view --minify
kubectl cluster-info
kubectl get nodes --request-timeout=10s

# Direct API server health check
API_SERVER=$(kubectl config view -o jsonpath='{.clusters[0].cluster.server}')
curl -sk ${API_SERVER}/healthz
curl -sk ${API_SERVER}/livez
curl -sk ${API_SERVER}/readyz
curl -sk ${API_SERVER}/version

# Check etcd directly
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status --write-out=table

# Check disk space (etcd killer)
df -h /var/lib/etcd

# Check control plane resources
systemctl status kubelet
crictl ps
crictl pods

# Check API server metrics
curl -sk ${API_SERVER}/metrics | grep apiserver_request_duration_seconds

# If using kubeadm
kubeadm certs check-expiration
kubeadm init phase certs all --dry-run
```

## Root Cause

| Root Cause | Indicators | Fix |
|---|---|---|
| Certificate expiration | TLS handshake errors in logs | `kubeadm certs renew all` and restart static pods |
| etcd disk full or slow | etcd high latency, WAL sync timeouts |扩容 etcd disk, move to SSD, compact/defrag |
| etcd member down | etcd cluster unhealthy | Recover etcd member or add new one |
| Control plane node down | Node NotReady, kubelet stopped | Restart kubelet, check node health |
| Load balancer misconfigured | LB health checks failing | Fix health check endpoint to `/healthz` |
| Resource exhaustion | High CPU/memory on control plane | Scale control plane, increase limits |
| Network ACL blocking | Security groups/NACLs blocking 6443 | Update network rules |
| DNS failure | Can't resolve API server hostname | Fix CoreDNS or use IP directly |

## Immediate Mitigation

1. **Use kubectl proxy from a control plane node**:
   ```bash
   ssh control-plane-1
   kubectl proxy --port=8001 &
   # Access via local:8001
   ```

2. **SSH tunnel to API server**:
   ```bash
   ssh -L 6443:localhost:6443 control-plane-1
   # Use kubeconfig pointing to localhost:6443
   ```

3. **Restart the API server** if it's crashed:
   ```bash
   sudo crictl stop $(sudo crictl ps -q --name kube-apiserver)
   # Kubelet will restart the static pod
   ```

4. **If etcd is the issue**, restart it:
   ```bash
   sudo crictl stop $(sudo crictl ps -q --name etcd)
   ```

## Permanent Fix

1. **Implement etcd monitoring** — monitor disk usage, WAL fsync duration, leader changes
2. **Use dedicated etcd nodes** — separate etcd from API server for resource isolation
3. **Certificate auto-rotation** — set up cert-manager or kubeadm certificate renewal automation
4. **API server HA** — run 3+ API servers behind a load balancer
5. **etcd backup automation** — schedule regular etcd snapshots:
   ```bash
   ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%Y%m%d).db
   ```
6. **Resource quotas on control plane** — ensure control plane nodes have sufficient resources
7. **Runbook and documentation** — document the recovery steps for the team

## Monitoring

```yaml
# Prometheus alert for API server health
- alert: KubernetesAPIServerDown
  expr: up{job="kubernetes-apiservers"} == 0
  for: 1m
  labels:
    severity: critical

- alert: APIServerLatencyHigh
  expr: histogram_quantile(0.99, rate(apiserver_request_duration_seconds_bucket[5m])) > 5
  for: 5m
  labels:
    severity: warning

- alert: APIServerRequestErrors
  expr: rate(apiserver_request_total{code=~"5.."}[5m]) > 0.05
  for: 5m
  labels:
    severity: critical
```

Monitor etcd specifically:
```bash
# etcd metrics to watch
etcd_disk_wal_fsync_duration_seconds
etcd_disk_backend_commit_duration_seconds
etcd_server_leader_changes_seen_total
etcd_network_peer_round_trip_time_seconds
```

## Security

- Never share kubeconfig files — use short-lived tokens (OIDC, webhook tokens)
- Ensure API server is only accessible via private networks or VPN
- Enable audit logging on the API server to track who did what
- Use RBAC to limit who can restart pods or modify cluster components
- Encrypt etcd at rest with KMS provider
- Rotate certificates regularly and automate with cert-manager

## Production Considerations

- **HA**: Run at least 3 API servers across 3 AZs with an LB
- **etcd**: Run 3 or 5 etcd nodes (odd numbers for quorum)
- **Backups**: Daily etcd snapshots stored in S3 with point-in-time recovery
- **Disaster Recovery**: Practice etcd restore quarterly
- **Cost**: Control plane node sizing — don't under-provision
- **Compliance**: Audit logging for SOC2/HIPAA requirements
- **Operational**: Document control plane access procedures for on-call

## Senior-Level Answer

"The API server being unreachable doesn't mean workloads are affected — pods continue running via kubelet's local control loop. My first step is to determine scope: is it my kubeconfig, network, or the actual API server? I check `/healthz` and `/readyz` endpoints directly, then SSH into a control plane node to inspect control plane pods and etcd health. Common causes are certificate expiration, etcd performance issues, or load balancer misconfiguration. I restore access via SSH tunnel or kubectl proxy, then implement fixes like cert renewal or etcd recovery. To prevent recurrence, I'd set up API server health monitoring, automated etcd backups, and certificate auto-rotation."

## Architect-Level Answer

"This incident reveals that the management plane is a single point of failure for operational capabilities, even though data plane is resilient. The architecture should have: (1) multi-AZ API server deployment with automatic failover behind an NLB, (2) dedicated etcd cluster across AZs with monitoring on WAL fsync latency and disk I/O, (3) automated certificate rotation via cert-manager, (4) etcd backup to S3 with automated restore testing, (5) infrastructure-as-code for the entire control plane to enable rapid rebuild. I'd also implement a 'break glass' procedure — a documented and tested way to access the cluster when the primary access path fails. For long-term strategy, consider managed Kubernetes (EKS/GKE) where the control plane is managed by the cloud provider."

## Follow-Up Questions

1. "How would you recover an etcd cluster if 2 out of 5 members are permanently lost?"
2. "What's the difference between `kubectl proxy` and `kubectl port-forward` in terms of how they connect to the API server?"
3. "How do you handle certificate rotation in a production cluster without downtime?"
4. "If the API server metrics show `etcd request latency > 5s`, how do you determine if it's etcd or the API server itself?"
5. "Design a disaster recovery plan for a Kubernetes cluster — what's your RTO and RPO strategy?"
