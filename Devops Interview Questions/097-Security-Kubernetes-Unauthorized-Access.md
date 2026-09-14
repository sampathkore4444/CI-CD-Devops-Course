# 97. Unauthorized Access to Kubernetes Cluster

## Scenario

Your monitoring team detects unexplained CPU spikes across several nodes in the Kubernetes cluster. Investigation reveals cryptocurrency mining pods running in the `default` namespace. Someone gained access to the Kubernetes API server using a compromised kubeconfig. The malicious pods are `kube-miner-*` in the `default` namespace, using Nvidia GPUs and high CPU. You need to detect the breach, evict the malicious pods, and harden the cluster. The cluster is EKS (Amazon) with 12 worker nodes.

## Interviewer Question

"Someone gained access to the Kubernetes API server using a compromised kubeconfig. They created pods that are mining cryptocurrency. How do you detect the breach, evict the malicious pods, and harden the cluster?"

## What I Should Think About

- Detection: audit logs (EKS control plane / API server audit), Falco runtime alerts, unusual workloads, node CPU spikes
- Order of operations: contain without tipping off the attacker; preserve evidence
- Evicting pods: delete workloads + prevent re-creation (namespaces, resource quotas, labels, taints, PSP/PSS)
- Finding the entry vector: which kubeconfig? which user? which API?
- Harden: RBAC least privilege, kubeconfig rotation, short-lived creds, OIDC, audit policy, NetworkPolicy, private nodes, EKS best practices
- Securing the attack surface: exposed kubelet, public API endpoint, blocked worker node access
- Kill switch: use labels/namespaces to identify + delete all related resources (Deployment/ReplicaSet/DaemonSet/StatefulSet/Jobs/Secrets)
- Keep evidence: audit logs, pod specs, node forensics, artifacts (miner images)
- Remediate vulnerability that allowed entry

## Ideal Answer

**Phase 1 — Detect & scope:**
1. Review API server audit logs for `kube-miner*` creation time and actor identity
2. Correlate: which kubeconfig/user/service account created them?
3. Enumerate all attacker-created resources (not just pods — they may have created namespaces, Roles, ClusterRoles, Secrets, or modified existing ones)
4. Check what other clusters share the same kubeconfig/identity

**Phase 2 — Contain:**
5. Revoke/kill the compromised credential immediately (rotate the kubeconfig, invalidate the identity, revoke IAM role/user / SA token)
6. Delete the malicious workloads; add deny rules + resource quotas for `default` namespace
7. Block `default` namespace from creating anything not labeled `app=trusted` via admission policy (Kyverno) — prevent churn

**Phase 3 — Root cause fix:**
7. Find how the kubeconfig leaked (GitHub? CI secrets? shared workstation?)
8. Harden cluster: RBAC least privilege, restrict `default` namespace, add Pod Security Standards (restricted), NetworkPolicies, Disable auto-mount SA token, consider admission webhooks enforcing image allow-list
9. Rotate ALL potentially exposed creds/service account tokens; rotate node/instance roles
10. Enable / tighten audit logging + Falco

**Phase 4 — Post-incident:**
11. Preserve artifacts (log events, pod manifests, node forensics) for legal/IP holder
12. Document runbook + lessons learned; purple-team the fix

## Architecture

```
  ATTACK FLOW:
  ┌────────────┬──────────────────────────┐  ┌───────────────────────┐
  │ Attacker   │ kubeconfig (leaked)      │ │ EKS API Server         │
  │            │  user: dev-admin         │ │  - leaked auth         │
  │            │  (compromised)           │ │  - RBAC permits create │
  └─────┬──────┴──────────────────────────┘  └──────────┬────────────┘
        │                                              │ create Pods
        │                                              ▼
        │                                  ┌─────────────────────────┐
        │                                  │ Pods: kube-miner-xxxx   │
        │                                  │ image: xmrig            │
        │                                  │ nodeSelector: GPU nodes │
        │                                  │ exec crypto-mining       │
        └──────────────────────────────────▶ CPU spikes across nodes │
                                            └─────────────────────────┘
  DETECTION: ALERT → api.audit RBAC actor = dev-admin
             falco: execve / process mining
             CPUUtilization node alert (miner workload)
```

## Investigation

**Step 1: Identify the malicious resources**
```bash
kubectl get pods -A -o wide | grep -iE "kube-miner|miner|xmrig|cryptonight"
kubectl get deploy,rs,ds,sts,job,cronjob -A | grep -i miner
kubectl get svc,configmaps,secrets -A | grep -iE "miner|malware"

# Check for suspicious namespaces
kubectl get ns | grep -iE "miner|dev|staging|temp"
```

**Step 2: Scrape the attacker's actual objects and images**
```bash
kubectl get pod kube-miner-abc123 -o yaml > /tmp/evidence-pod.yaml
kubectl describe pod kube-miner-abc123 | tail -40
kubectl get events -n default | grep -i miner | tail -30

# check image
kubectl get pod kube-miner-abc123 -o jsonpath='{.spec.containers[0].image}'
# → attacker repo / xmrig:latest
```

**Step 3: Audit logs — find the actor & timeline**
```bash
# EKS Audit Logs (CloudWatch) enabled?
aws eks describe-cluster --name prod --query 'cluster.logging'
# Search audit for Pod create around breach time
aws logs filter-log-events --log-group-name /aws/eks/prod/cluster/audit \
  --filter-pattern 'kube-miner' | jq .events | head
# Identify user HTTP dictionary:
aws logs filter-log-events ... --filter-pattern '"user.username":"dev-admin"'

# find which token/IP; look for unusual actor
aws logs ... | jq .events[0].message | jq .user
# check verb/response: create/exec/list/delete on which resources
```

**Step 4: Check for privilege escalation / lateral movement**
```bash
kubectl get clusterroles,roles -A | grep -iE "admin|dev"
kubectl get rolebindings,clusterrolebindings -A -o yaml | grep -B2 -A5 dev-admin
# confirm nothing else was modified:
kubectl get cronjobs -A   # persistence mechanism?
kubectl get secrets -A   # check no new root/service account secrets
```

**Step 5: Node forensics**
```bash
kubectl debug node/<node> -it --image=ubuntu
# or collect node logs:
# docker/containerd: docker ps -a | grep miner
# files under /tmp, cron entries in node crontab
jcrontab -l | grep -i xmrig
# network egress: lsof -i / netstat -tnp
```

## Commands

```bash
# 1. Kill attackers' pods/compute (but ALSO delete the controller so they don't respawn)
kubectl delete deploy,rs,ds,sts,job,cronjob,pod -n default \
  -l app=kube-miner --wait=false
kubectl delete ns <suspicious-ns> --wait=false

# 2. Kill the compromised identity NOW
#   - Revoke kubeconfig / user in OIDC, or:
kubectl delete secret <user-sa-token> -n <ns>
#   - AWS: rotate IAM creds of user dev-admin
aws iam delete-access-key --user-name dev-admin --access-key-id <LEAKED>
aws iam create-access-key --user-name <new>
#   - revoke the ClusterRoleBinding
kubectl delete clusterrolebinding dev-admin-binding

# 3. Prevent re-creation (lethal-default):
kubectl apply -f - << 'EOF'
apiVersion: v1
kind: ResourceQuota
metadata: {name: default-quota, namespace: default}
spec:
  hard:
    pods: "500"
    cpu: "100"
EOF

kubectl apply -f - << 'EOF'
apiVersion: v1
kind: LimitRange
metadata: {name: default-limit, namespace: default}
spec:
  limits:
    - max: {cpu: "4", memory: 8Gi}   # cap node hogging
      default: {cpu: 500m, memory: 256Mi}
EOF

# 4. Blockty admission (Kyverno): deny anything not from trusted image
kubectl apply -f deny-mining.yaml   # matches image repo patterns (xmrig/cryptonight)

# 5. Global lockdown:
kubectl get nodes -o name | xargs -I{} kubectl taint nodes {} \
  dedicated=blocked:NoSchedule
# (re-allow: kubectl taint nodes ... -)

# 6. Recreate worker nodes if attacker has node access:
aws eks... nodegroup replace; or kubectl drain + cordon all miners' nodes
```

## Root Cause

1. **Leaked kubeconfig** (CI secrets / workstation / repo .kube) — kubeconfig had admin privileges; keep kubeconfigs in a vault
2. **Over-privileged RBAC:** `cluster-admin` binding for an entire namespace → attacker got `create pod` everywhere
3. **No runtime detection** in the loop (Falco not installed / no CPU-spike alert)
4. **No namespace isolation / network policy** allowing broad egress (miner needs external pool)
5. **Public/loose node access**: worker nodes reachable / image pull allowed from anywhere

## Immediate Mitigation

```bash
# 1. DELETE attacker pods + controllers
kubectl delete deploy,rs,ds,sts,job,cronjob,pod -n default \
  -l app=kube-miner --force --grace-period=0

# 2. REVOKE kubeconfig immediately (block further API calls)
#   - disable IAM user / delete token, rotate

# 3. Use taint / cordon on affected GPU nodes
kubectl cordon <node1> <node2>   # stop scheduling

# 4. Add admission deny (Kyverno)
# 5. Ensure no new pods: temporarily disable auto-rollout of any workloads
# 6. Preserve evidence before deleting nodes (pod.yaml, audit logs)
```

## Permanent Fix

1. **RBAC least privilege**: per-namespace service accounts with explicit roles; never `cluster-admin`
2. **Short-lived credentials**: OIDC + IRSA/IAM roles; reduce kubeconfig rotation burden
3. **Admission policy (Kyverno/OPA/Gatekeeper)**:
   - Image allow-list/deny-list, restrict `default` namespace, enforce PSS restricted
   - Auto-approve nothing; require `app` labels
4. **NetworkPolicy** default deny egress; allow only service-to-service
5. **Pod Security Standards**: restricted mode for all namespaces
6. **Runtime detection**: Falco daemonset with miner/exec anomalies; GKE/EKS audit events to SIEM
7. **API blocking**: IP allowlist/restrict API public; use IAM condition on kubeconfig origins
8. **Automated secret scanning** and Vault for kubeconfigs, rotating node SSH keys
9. **Image registry admission**: only pull from trusted registry (Olive/ECR with policy verification)

## Monitoring

```bash
# EKS/control-plane:
# - Audit log events: create/exec pod with unusual actor → alert
# - kubeconfig rotation alerts, failed auth spikes (brute force)
# Runtime (Falco):
# - execve into container we don't own, reverse shell, mining binaries
# Pod/node:
# - CPUUtilization anomaly (miner spike)
# - image not in approved registry/allow-list → warning
# Network:
# - egress to known mining pool hostnames/C2 IPs → P1
# IDS/GPG: Wazuh/Falco anomaly; nodeagent: process hash hits virustotal
```

## Security

- Assume the breached credential persists in copies; rotate everything it could touch (SA tokens, node role ARNs, cluster CA? only if needed)
- Network chokepoint: block non-allowlisted egress (miner can't phone home)
- Keep evidence for forensics: pod yaml, audit events, node memory dump (containers), SIEM preserves logs
- Minimize standing "break-glass" kubeconfigs; require MFA for any admin access
- Consider moving worker nodes to private subnets (no public IP); restrict image pull to registry

## Production Considerations

- **HA**: cordoning/draining nodes costs capacity — time rollout carefully
- **Reliability**: a miner hogging GPUs starves real workloads; bind GPU nodes to trusted labels/taints
- **Cost**: crypto-mining on your compute = large AWS bill — alert on node CPU sustained > 90%
- **Compliance**: breach of production workload = potential data-residency issue; preserve <anything> for DPIA
- **Operational**: runbook for cluster compromise — incident commander, evidence freeze, containment order, communication
- **Governance**: least-privilege RBAC review quarterly; restrict `default` namespace; enforce Pod Security Admission via policy-as-code

## Senior-Level Answer

"First, I'd use audit logs to identify the exact actor and timeline, then enumerate ALL resources they created — pods, Crontabs (persistence), Roles, and Secrets — not just the visible miner batch. Contain: delete the workloads AND delete the compromised kubeconfig identity / rotate the token, then add deny admission + quotas so nothing respawns. Harden: least-privilege RBAC, restrict `default` namespace, NetworkPolicy default-deny egress, and Falco for runtime detection. The core fix is making creation-by-unknown-use impossible: admission policies, image allow-lists, PSS restricted, and no standing cluster-admin kubeconfigs. Every EKS cluster in the org should get the same audit-log-to-SIEM pipeline and taint-based node isolation to prevent a repeat."

## Architect-Level Answer

"The lesson: a single kubeconfig shouldn't be able to deploy anywhere. Architecturally I'd adopt multi-layer controls: (1) identity — OIDC + short-lived tokens + IRSA, no static kubeconfig for humans; (2) authn/z — least-privilege RBAC + policy-as-code admission (Kyverno) so arbitrary pod creation is impossible; (3) isolation — per-team namespaces with quotas, taints for specialized (GPU) nodes, NetworkPolicies default-deny; (4) runtime — Falco triggers on mining/exec/reverse-shell; (5) detection — audit to SIEM with actions (create pod, exec) flagged. Worker nodes in private subnets, image pulls restricted to a verified registry produce workloads with Sigstore/cosign. Combine that with a cluster-compromise runbook that says rotate identity first, freeze evidence second, evict third — and this class of breach becomes a contained PSIRT-style event, not a multi-day fire."

## Follow-Up Questions

1. "How do you distinguish a real mining pod from a legit heavy workload without relying solely on image names?"
2. "If the leaked kubeconfig outlives delivery, what's your strategy to make its compromise cheap for every cluster?"
3. "How does `automountServiceAccountToken: false` and IRSA change this attack path?"
4. "Design a Falco rule set that catches xmrig/cryptonight behavior without false positives on CI/CD builds."
5. "What forensic artifacts must you preserve for the first 4 hours after discovery without breaking production?"