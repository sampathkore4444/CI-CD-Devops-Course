# 59. Network Policy Blocking Legitimate Traffic

## Scenario

You just applied a new NetworkPolicy to the `production` namespace as part of a security hardening initiative. The policy was supposed to restrict inter-service communication to only the explicitly allowed paths. Ten minutes later, the frontend can't reach the API, the API can't reach the database, and the order service can't reach the payment service. Users are seeing 502 errors. The NetworkPolicy was applied by the security team without testing in staging. You need to quickly identify which rules are blocking traffic and fix them without removing all security controls.

## Interviewer Question

"After applying a new NetworkPolicy, inter-service communication broke. How do you debug NetworkPolicy rules, identify what's being blocked, and fix the issue while maintaining security?"

## What I Should Think About

- NetworkPolicies are additive — they only restrict, never allow beyond default
- Default behavior depends on CNI: some CNIs (Calico, Cilium) enforce by default, others (flannel) don't enforce at all
- NetworkPolicies are additive within a namespace — all policies apply
- A policy with empty ingress/egress means NO traffic is allowed
- Label selectors must match exactly — a typo in labels can cause issues
- NetworkPolicies require a CNI that supports them — not all do
- DNS (kube-dns/coredns) must be explicitly allowed in egress rules
- Debugging involves checking both ingress and egress rules from both sides

## Ideal Answer

**Step 1: Identify the problematic policy**

```bash
# List all NetworkPolicies in the namespace
kubectl get networkpolicy -n production
kubectl describe networkpolicy <policy-name> -n production

# Check which pods are affected
kubectl get pods -n production --show-labels

# Look at the policy details
kubectl get networkpolicy <policy-name> -n production -o yaml
```

**Step 2: Understand the policy rules**

```yaml
# Example of a problematic policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-all-traffic
  namespace: production
spec:
  podSelector: {}  # Applies to ALL pods
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend  # Only allow from frontend
    ports:
    - port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          role: api  # Only allow to API
    ports:
    - port: 8080
```

**Step 3: Debug traffic flow**

```bash
# Test connectivity from frontend to API
kubectl exec -it <frontend-pod> -n production -- curl -v http://api-service:8080/healthz

# Check if DNS is working (common issue with egress rules)
kubectl exec -it <frontend-pod> -n production -- nslookup api-service.production.svc.cluster.local
kubectl exec -it <frontend-pod> -n production -- cat /etc/resolv.conf

# Check if DNS is allowed in egress rules
# Most NetworkPolicy egress rules need to explicitly allow DNS (port 53)

# Test from a debug pod with no labels (worst case)
kubectl run debug-netpol --image=nicolaka/netshoot -n production --rm -it --restart=Never -- bash
# Inside debug pod:
nslookup api-service.production.svc.cluster.local
curl -v http://api-service:8080/healthz
```

**Step 4: Fix the policy**

```bash
# Option 1: Remove the problematic policy
kubectl delete networkpolicy restrict-all-traffic -n production

# Option 2: Fix the policy to allow necessary traffic
# Add DNS egress rule
# Add rules for each service pair
```

**Step 5: Verify the fix**

```bash
# Test all service-to-service communication
kubectl exec -it <frontend-pod> -n production -- curl -s http://api-service:8080/healthz
kubectl exec -it <api-pod> -n production -- curl -s http://db-service:3306/healthz
kubectl exec -it <order-pod> -n production -- curl -s http://payment-service:8080/healthz
```

## Architecture

```
    Network Policy Traffic Flow
    ───────────────────────────

    BEFORE Policy (all traffic allowed):
    ┌──────────┐    ┌──────────┐    ┌──────────┐
    │ Frontend │───▶│   API    │───▶│Database  │
    │          │    │          │    │          │
    └──────────┘    └──────────┘    └──────────┘
         │              │               │
         │              ▼               │
         │         ┌──────────┐         │
         └────────▶│ Payment  │◀────────┘
                   └──────────┘

    AFTER Restrictive Policy:
    ┌──────────┐    ┌──────────┐    ┌──────────┐
    │ Frontend │──X─│   API    │──X─▶│Database  │
    │          │    │          │    │          │
    └──────────┘    └──────────┘    └──────────┘
         │              │               │
         │              ▼               │
         │         ┌──────────┐         │
         └────X───▶│ Payment  │◀──X─────┘
                   └──────────┘

    X = Blocked by NetworkPolicy

    Issues Found:
    1. DNS (port 53) not allowed in egress → API can't resolve DB hostname
    2. API→DB not in ingress rules → DB blocks API traffic
    3. Order→Payment not in either rule set
    4. Health checks from kube-proxy blocked
```

## Investigation

1. **List all NetworkPolicies** in the namespace — check what was recently added
2. **Check policy selectors** — podSelector and namespaceSelector labels must match
3. **Check egress rules** — DNS (port 53) must be explicitly allowed
4. **Check ingress rules** — each service pair needs explicit ingress/egress rules
5. **Test with a debug pod** — use nicolaka/netshoot for network debugging
6. **Check CNI** — not all CNIs enforce NetworkPolicies (flannel doesn't)
7. **Check policy types** — `policyTypes: [Ingress, Egress]` means both directions restricted
8. **Check for empty rules** — empty ingress/egress means no traffic allowed
9. **Check labels** — labels must match exactly between pods and policy selectors
10. **Check kube-dns/CoreDNS** — it needs egress access to upstream DNS

## Commands

```bash
# List all NetworkPolicies
kubectl get networkpolicy -A
kubectl get networkpolicy -n production -o yaml

# Check which pods are selected by policies
kubectl get pods -n production -o json | jq '.items[] | {name: .metadata.name, labels: .metadata.labels}'

# Test DNS resolution
kubectl exec -it debug-netpol -n production -- nslookup kubernetes.default.svc.cluster.local
kubectl exec -it debug-netpol -n production -- dig +short api-service.production.svc.cluster.local

# Test connectivity
kubectl exec -it debug-netpol -n production -- curl -v --connect-timeout 5 http://api-service:8080/healthz
kubectl exec -it debug-netpol -n production -- nc -zv api-service 8080
kubectl exec -it debug-netpol -n production -- nc -zv db-service 3306

# Check CNI plugin
kubectl get pods -n kube-system -o wide | grep -E "calico|cilium|flannel|weave"

# Check if CNI supports NetworkPolicy
kubectl get crd | grep networkpolicy
```

## Root Cause

| Root Cause | Symptom | Fix |
|---|---|---|
| DNS not allowed in egress | Services can't resolve hostnames | Add egress rule for port 53 to kube-dns |
| Empty ingress rules | All ingress blocked | Add explicit ingress rules for each service pair |
| Empty egress rules | All egress blocked | Add explicit egress rules for each destination |
| Label mismatch | Policy doesn't affect intended pods | Fix label selectors to match pod labels |
| CNI doesn't enforce | Policy applied but no effect | Install CNI that supports NetworkPolicy |
| Namespace selector wrong | Cross-namespace traffic blocked | Fix namespaceSelector labels |
| Health check blocked | Liveness/readiness probes fail | Allow traffic from kube-proxy or node |

## Immediate Mitigation

```bash
# Option 1: Delete the problematic policy
kubectl delete networkpolicy <policy-name> -n production

# Option 2: Apply a permissive policy to allow all traffic temporarily
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-temp
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - {}
  egress:
  - {}
EOF

# Option 3: Scale down the affected pods (if you can't fix the policy)
# This is a last resort
```

## Permanent Fix

1. **Test policies in staging first** — always test NetworkPolicy changes
2. **Start with egress-only policies** — easier to debug and less disruptive
3. **Always allow DNS** — add egress rule for port 53 to kube-dns:
   ```yaml
   egress:
   - to:
     - namespaceSelector:
         matchLabels:
           kubernetes.io/metadata.name: kube-system
       podSelector:
         matchLabels:
           k8s-app: kube-dns
     ports:
     - port: 53
       protocol: UDP
     - port: 53
       protocol: TCP
   ```
4. **Use namespace isolation gradually** — don't lock everything down at once
5. **Document traffic flows** — maintain a matrix of which services talk to which
6. **Implement gradually** — apply policies per service, not all at once

## Monitoring

```yaml
# Alert on NetworkPolicy changes
- alert: NetworkPolicyApplied
  expr: kube_networkpolicy_created > 0
  for: 1m
  labels:
    severity: info

# Monitor dropped connections
- alert: HighConnectionDrops
  expr: rate(container_network_tcp_usage_total{tcp_state="drop"}[5m]) > 100
  labels:
    severity: warning
```

Monitor network policies:
```bash
# Check policy enforcement status (Cilium)
kubectl get cnp -n production
cilium policy get

# Check Calico policy status
calicoctl get networkpolicy
```

## Security

- NetworkPolicies are essential for zero-trust networking
- Default deny policy should be applied to all namespaces
- Allow only necessary traffic flows
- Regular audit of NetworkPolicy rules
- Test all policy changes in staging before production
- Monitor for policy violations

## Production Considerations

- **HA**: NetworkPolicies should allow health check traffic
- **Reliability**: DNS must always be accessible — add explicit egress rules
- **Security**: Default deny + explicit allow is the gold standard
- **Operational**: Document all NetworkPolicy rules for the team
- **Compliance**: PCI/HIPAA require network segmentation — NetworkPolicies help
- **Cost**: No direct cost, but complexity increases operational overhead

## Senior-Level Answer

"NetworkPolicies are additive — they only restrict, never allow beyond default. When I see broken inter-service communication after applying a policy, I first check what the policy actually allows. The most common issue is forgetting to allow DNS (port 53) in egress rules — services can't resolve hostnames. I use a debug pod with nicolaka/netshoot to test connectivity and DNS. I check if the CNI actually enforces NetworkPolicies (flannel doesn't). For the fix, I'd either remove the problematic policy or fix it to allow necessary traffic. Long-term, I'd implement a default-deny policy per namespace and add allow rules incrementally, testing each change."

## Architect-Level Answer

"NetworkPolicy management requires a systematic approach. The architecture should be: (1) namespace-level default-deny policies for both ingress and egress, (2) per-service allow policies based on documented traffic flows, (3) a DNS egress rule in every namespace policy, (4) policy testing in staging with automated validation, (5) CNI selection that supports full NetworkPolicy enforcement (Cilium or Calico over flannel), (6) policy-as-code with CI/CD validation. I'd maintain a service dependency matrix that maps all allowed traffic flows and automatically generates NetworkPolicies from it. For compliance, this also provides audit evidence of network segmentation."

## Follow-Up Questions

1. "How do NetworkPolicies interact with Services of type NodePort or LoadBalancer?"
2. "What happens if two NetworkPolicies select the same pod with conflicting rules?"
3. "How would you implement namespace-level isolation while still allowing cross-namespace service communication?"
4. "Explain the difference between Calico, Cilium, and Flannel in terms of NetworkPolicy enforcement."
5. "How do you test NetworkPolicy changes safely in production without risking downtime?"
