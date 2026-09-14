# 46. Pod Running but Application Is Unreachable

## Scenario

The `payment-gateway` service in the `ecommerce` namespace shows 2/2 containers Running. The deployment status looks healthy. However, users are reporting that payment processing is failing with "connection refused" errors. The service object exists with correct selector labels. You can see endpoints registered in the Service. But when you curl the service ClusterIP from another pod, you get no response. The ingress is also returning 502 Bad Gateway. The service was working 2 hours ago before a network policy change was applied.

## Interviewer Question

"kubectl get pods shows the pod is Running (2/2). But users report the application is not accessible. How do you trace the issue from pod → container → readiness probe → service → endpoints → ingress → DNS?"

## What I Should Think About

- "Running" status only means the containers are executing — it doesn't mean the application is healthy or the network path is working
- Must trace the full path: Pod → Container Port → Readiness Probe → Service → Endpoints → Ingress → DNS
- Network Policies are a common culprit — they're applied but often forgotten
- The readiness probe might be failing, causing the pod to be removed from Endpoints
- The Service selector might not match pod labels
- Ingress controller might have its own issues (backend connectivity, configuration)
- DNS resolution inside the cluster could be broken
- Must check kube-proxy and iptables rules

## Ideal Answer

"Start at the bottom of the stack and work up. First verify the container itself is serving:

```bash
kubectl exec -it <pod> -n ecommerce -c payment-gateway -- curl -s localhost:8443/health
```

If that works, check if the pod is in the Service's endpoints:

```bash
kubectl get endpoints payment-gateway -n ecommerce
```

If endpoints are empty, the readiness probe is likely failing. Check:

```bash
kubectl describe pod <pod> -n ecommerce | grep -A 5 "Readiness"
```

If endpoints show the pod IP, test the Service directly:

```bash
kubectl exec -it <debug-pod> -n ecommerce -- curl -s payment-gateway.ecommerce.svc.cluster.local:8443/health
```

If that fails, check NetworkPolicies:

```bash
kubectl get networkpolicy -n ecommerce -o wide
```

If that looks correct, check kube-proxy and iptables on the node. Finally, if it's an external issue, check the Ingress controller logs and verify DNS resolution from outside the cluster."

## Architecture

```
User Request Flow:

External Traffic
      │
      ▼
┌─────────────┐     ┌──────────────────┐     ┌─────────────────┐
│   Ingress    │────▶│  Ingress         │────▶│  Service        │
│   Resource   │     │  Controller      │     │  ClusterIP      │
│              │     │  (Nginx/Traefik) │     │  :8443          │
└─────────────┘     └──────────────────┘     └────────┬────────┘
                                                      │
                         ┌────────────────────────────┘
                         │  kube-proxy / iptables rules
                         ▼
              ┌─────────────────────┐
              │   Service Endpoints │
              │   (Pod IPs:port)    │
              └─────────┬──────────┘
                        │
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
   ┌──────────┐  ┌──────────┐  ┌──────────┐
   │  Pod A   │  │  Pod B   │  │  Pod C   │
   │ 10.0.1.5 │  │ 10.0.1.6 │  │ 10.0.1.7 │
   │ :8443    │  │ :8443    │  │ :8443    │
   └──────────┘  └──────────┘  └──────────┘
         ▲
         │
    ┌────┴────────────────────┐
    │  NetworkPolicy          │
    │  - Ingress rules        │
    │  - Egress rules         │
    │  - Pod selector         │
    └─────────────────────────┘

Problem Points:
  1. Ingress → Controller: misconfigured backend
  2. Controller → Service: wrong path or port
  3. Service → Endpoints: readiness probe failing
  4. Endpoints → Pod: NetworkPolicy blocking
  5. Pod container: application crashed but container running
  6. DNS: service name not resolving
```

## Investigation

**Step 1: Verify the container is actually serving traffic**

```bash
kubectl exec -it <pod-name> -n ecommerce -c payment-gateway -- wget -qO- http://localhost:8443/health
```

**Step 2: Check if pod is in Service endpoints**

```bash
kubectl get endpoints payment-gateway -n ecommerce
```

Expected output (healthy):
```
NAME              ENDPOINTS                                      AGE
payment-gateway   10.244.1.5:8443,10.244.2.8:8443,10.244.3.2:8443   2h
```

If endpoints are empty or missing pod IPs, the readiness probe is failing.

**Step 3: Check readiness probe status**

```bash
kubectl get pod <pod-name> -n ecommerce -o jsonpath='{range .status.conditions[?(@.type=="Ready")]}{.status}{"\t"}{.message}{"\n"}{end}'
```

**Step 4: Test Service from inside the cluster**

```bash
kubectl run debug --image=curlimages/curl --rm -it --restart=Never -n ecommerce -- \
  curl -v --connect-timeout 5 payment-gateway.ecommerce.svc.cluster.local:8443/health
```

**Step 5: Check NetworkPolicies**

```bash
kubectl get networkpolicy -n ecommerce -o yaml
```

**Step 6: Check Ingress configuration and controller logs**

```bash
kubectl get ingress payment-gateway -n ecommerce -o yaml
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx --tail=50
```

**Step 7: Verify DNS resolution**

```bash
kubectl run dns-test --image=busybox --rm -it --restart=Never -- nslookup payment-gateway.ecommerce.svc.cluster.local
```

**Step 8: Check iptables rules on the node**

```bash
kubectl debug node/<node-name> -it --image=busybox -- iptables -t nat -L KUBE-SERVICES | grep payment
```

## Commands

```bash
# Full network path trace
# 1. Can we reach the container directly?
kubectl exec -it <pod-name> -n ecommerce -c payment-gateway -- wget -qO- http://localhost:8443/health

# 2. Are endpoints populated?
kubectl get endpoints payment-gateway -n ecommerce -o yaml

# 3. Does the Service selector match pod labels?
kubectl get svc payment-gateway -n ecommerce -o jsonpath='{.spec.selector}'
kubectl get pods -n ecommerce -l app=payment-gateway --show-labels

# 4. Test Service from another pod
kubectl run test-pod --image=curlimages/curl --rm -it --restart=Never -n ecommerce -- \
  curl -v http://payment-gateway.ecommerce.svc.cluster.local:8443/health

# 5. DNS resolution test
kubectl run dns-test --image=busybox --rm -it --restart=Never -- \
  nslookup payment-gateway.ecommerce.svc.cluster.local

# 6. Check all NetworkPolicies in the namespace
kubectl get networkpolicy -n ecommerce -o wide
kubectl describe networkpolicy -n ecommerce

# 7. Check if any NetworkPolicy selects this pod
kubectl get networkpolicy -n ecommerce -o json | jq '.items[] | select(.spec.podSelector.matchLabels.app=="payment-gateway")'

# 8. Verify Ingress backend connectivity
kubectl get ingress payment-gateway -n ecommerce -o jsonpath='{.spec.rules[*].http.paths[*]}'

# 9. Check Ingress controller logs for errors
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller --tail=100 | grep -i "payment\|502\|error"

# 10. Check kube-proxy health
kubectl get pods -n kube-system -l k8s-app=kube-proxy
kubectl logs -n kube-system -l k8s-app=kube-proxy --tail=50

# 11. Test external ingress endpoint
curl -v https://payment.example.com/health -H "Host: payment.example.com"

# 12. Check TLS certificate
kubectl get secret -n ecommerce tls-secret -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -text -noout
```

## Root Cause

| Root Cause | Evidence | How to Eliminate |
|---|---|---|
| **Readiness Probe Failing** | Endpoints empty, pod has `Ready=False` condition | Fix probe path, increase timeout, check app health endpoint |
| **NetworkPolicy Blocking** | NetworkPolicy exists with ingress/egress deny rules | Add allow rules for the service traffic |
| **Service Selector Mismatch** | Service selector doesn't match pod labels | Fix selector labels to match pod labels |
| **Ingress Misconfiguration** | Ingress backend service/port wrong, 502 in ingress logs | Correct the backend service reference |
| **DNS Resolution Failure** | `nslookup` fails for service name | Check CoreDNS pods, restart CoreDNS |
| **kube-proxy Broken** | iptables rules missing for the service | Restart kube-proxy, check its logs |
| **Container App Crashed** | App logs show crash, but container still running (restart count 0) | Check app logs, restart pod |
| **TLS Mismatch** | TLS termination at ingress fails | Verify certificate matches hostname |

## Immediate Mitigation

```bash
# 1. If NetworkPolicy is blocking, remove it temporarily
kubectl delete networkpolicy <policy-name> -n ecommerce

# 2. If readiness probe is failing, disable it temporarily to get traffic flowing
kubectl patch deployment payment-gateway -n ecommerce --type='json' -p='[
  {"op": "remove", "path": "/spec/template/spec/containers/0/readinessProbe"}
]'

# 3. If Service endpoints are empty, manually add pod IPs (emergency only)
kubectl patch svc payment-gateway -n ecommerce --type='json' -p='[
  {"op": "replace", "path": "/spec/externalIPs", "value": ["10.0.1.5"]}
]'

# 4. If Ingress is broken, use port-forward as temporary access
kubectl port-forward svc/payment-gateway 8443:8443 -n ecommerce &

# 5. Restart kube-proxy if iptables are corrupted
kubectl rollout restart daemonset kube-proxy -n kube-system
```

## Permanent Fix

```yaml
# 1. Fix the readiness probe
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-gateway
  namespace: ecommerce
spec:
  template:
    spec:
      containers:
      - name: payment-gateway
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8443
          initialDelaySeconds: 15
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3

---
# 2. Add proper NetworkPolicy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-payment-gateway-ingress
  namespace: ecommerce
spec:
  podSelector:
    matchLabels:
      app: payment-gateway
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    - podSelector:
        matchLabels:
          app: api-gateway
    ports:
    - protocol: TCP
      port: 8443
```

## Monitoring

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: payment-gateway-network
spec:
  groups:
  - name: network.rules
    rules:
    - alert: ServiceNoEndpoints
      expr: kube endpoints address{namespace="ecommerce",endpoint="payment-gateway"} == 0
      for: 2m
      labels:
        severity: critical
      annotations:
        summary: "Service payment-gateway has no endpoints"

    - alert: IngressHigh5xxRate
      expr: rate(nginx_ingress_controller_requests{status=~"5..",namespace="ecommerce"}[5m]) > 0.05
      for: 5m
      labels:
        severity: critical
```

## Security

- **Network Policies are critical**: Default deny all ingress/egress, then explicitly allow only required traffic.
- **Service mesh consideration**: Istio/Linkerd provide mTLS between services, preventing unauthorized pod-to-pod communication.
- **Ingress TLS**: Always enforce TLS termination. Use cert-manager for automatic certificate rotation.
- **Pod Security**: Restrict hostNetwork, hostPID, and hostIPC.

## Production Considerations

- **Multiple failure domains**: Test that losing one pod doesn't break the entire service path
- **Circuit breakers**: Implement circuit breakers to prevent cascade failures when one service is unreachable
- **Service mesh**: Consider Istio for observability and traffic management
- **Canary testing**: Test network policy changes in a canary namespace first

## Senior-Level Answer

"I'd trace the full network path from ingress to pod. Start by testing the container directly with `kubectl exec` — if the app responds on localhost, the container is healthy. Then check if the pod IP appears in the Service endpoints (kubectl get endpoints). Empty endpoints mean the readiness probe is failing. If endpoints are populated, test Service DNS resolution from another pod. If DNS works but traffic doesn't flow, check NetworkPolicies — a recently applied policy is likely blocking ingress. The fact that this broke 2 hours ago after a network policy change is the key indicator. I'd check the policy's podSelector and ingress rules, and verify that it allows traffic from the ingress controller namespace to the payment-gateway pods on port 8443. The fix is adding an ingress allow rule to the NetworkPolicy."

## Architect-Level Answer

"This scenario highlights the importance of network policy management as a first-class concern. At the architectural level, I'd implement: (1) A default-deny NetworkPolicy per namespace with explicit allow rules documented and version-controlled, (2) A staging environment that mirrors production NetworkPolicies to catch these issues before production, (3) Automated testing of network connectivity in the CI pipeline using tools like `netshoot` pods, (4) Service mesh (Istio/Linkerd) for mTLS and fine-grained traffic policies instead of raw NetworkPolicies, and (5) Observability: ensure every network policy change triggers a connectivity test. The key architectural principle is that network policies should be treated as code — reviewed, tested, and versioned like any other infrastructure change."

## Follow-Up Questions

1. "How do Kubernetes NetworkPolicies interact with Service mesh sidecar proxies?"
2. "What happens to existing connections when a NetworkPolicy is applied that denies traffic?"
3. "How would you debug a NetworkPolicy that allows traffic but the traffic is still being blocked?"
4. "Explain the difference between a NetworkPolicy with empty `podSelector` vs one that selects all pods."
5. "How does kube-proxy update iptables when a Service endpoint is added or removed?"
