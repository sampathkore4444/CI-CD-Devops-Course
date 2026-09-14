# 50. Service Not Routing Traffic to Pods

## Scenario

The `notification-service` in the `messaging` namespace has a Kubernetes Service with 3 endpoints listed in `kubectl get endpoints`. However, traffic monitoring shows only 1 pod is receiving requests. Users experience intermittent failures — approximately 66% of requests fail with connection timeouts. The Service uses ClusterIP type. The pods are running on 3 different nodes. The service selector matches all 3 pods. The application is a simple HTTP service with a health endpoint on port 8080. The issue started 1 hour ago after a node maintenance event.

## Interviewer Question

"A Kubernetes Service exists with 3 endpoints but traffic is only reaching 1 pod. Users experience intermittent failures. The service uses round-robin. How do you diagnose the load balancing and traffic routing issue?"

## What I Should Think About

- kube-proxy implements Service load balancing via iptables or IPVS rules
- If only 1 pod receives traffic, kube-proxy rules might be corrupted or not updated
- The node maintenance event might have disrupted kube-proxy on some nodes
- Endpoints might show 3 IPs but kube-proxy might not have all rules
- Check if pods are actually reachable from all nodes
- iptables vs IPVS mode matters for debugging
- Check if network policies are interfering
- Verify kube-proxy is running on all nodes
- The issue could be node-local: traffic from certain nodes only routes to the nearest pod

## Ideal Answer

"The fact that endpoints show 3 IPs but traffic only reaches 1 pod suggests a kube-proxy issue. kube-proxy maintains the Service-to-Pod mapping via iptables (or IPVS) rules on each node. After node maintenance, kube-proxy might not have re-synced its rules.

First, verify that all 3 endpoints are truly healthy:

```bash
kubectl get endpoints notification-service -n messaging -o yaml
```

Then test connectivity to each pod IP individually:

```bash
kubectl exec -it <test-pod> -n messaging -- curl http://10.244.1.5:8080/health
kubectl exec -it <test-pod> -n messaging -- curl http://10.244.2.8:8080/health
kubectl exec -it <test-pod> -n messaging -- curl http://10.244.3.2:8080/health
```

If all pods respond directly, the issue is in kube-proxy. Check kube-proxy pods and restart if needed.

## Architecture

```
Service Load Balancing (iptables mode):

  Client Pod                    kube-proxy iptables rules
      │
      │  Service ClusterIP: 10.96.45.123:8080
      │
      ▼
  ┌───────────────────────────────────────────────────┐
  │ iptables DNAT chain:                              │
  │                                                   │
  │  10.96.45.123:8080 → DNAT to:                     │
  │    ├── 10.244.1.5:8080   (probability: 0.333)    │
  │    ├── 10.244.2.8:8080   (probability: 0.333)    │
  │    └── 10.244.3.2:8080   (probability: 0.334)    │
  │                                                   │
  │  If rules are CORRUPTED:                          │
  │    └── Only 10.244.1.5:8080 (probability: 1.0)   │
  └───────────────────────────────────────────────────┘

Problem: kube-proxy rules on some nodes only point to 1 pod

  Node-1 (healthy):    Rules → Pod-1, Pod-2, Pod-3 ✓
  Node-2 (broken):     Rules → Pod-1 only ✗
  Node-3 (broken):     Rules → Pod-1 only ✗

  Result: Pods on Node-2 and Node-3 only reach Pod-1
          Pods on Node-1 load balance correctly
```

## Investigation

**Step 1: Verify endpoints match pods**

```bash
kubectl get endpoints notification-service -n messaging -o yaml
kubectl get pods -n messaging -l app=notification-service -o wide
```

**Step 2: Test direct pod connectivity**

```bash
kubectl exec -it <test-pod> -n messaging -- curl -s http://10.244.1.5:8080/health
kubectl exec -it <test-pod> -n messaging -- curl -s http://10.244.2.8:8080/health
kubectl exec -it <test-pod> -n messaging -- curl -s http://10.244.3.2:8080/health
```

**Step 3: Check kube-proxy status**

```bash
kubectl get pods -n kube-system -l k8s-app=kube-proxy
kubectl logs -n kube-system -l k8s-app=kube-proxy --tail=50
```

**Step 4: Verify iptables rules**

```bash
kubectl debug node/<node-name> -it --image=busybox -- iptables -t nat -L KUBE-SERVICES | grep notification
```

**Step 5: Check kube-proxy mode**

```bash
kubectl get cm kube-proxy -n kube-system -o yaml | grep mode
```

**Step 6: Test Service from multiple nodes**

```bash
# Run test pods on different nodes
kubectl run test-node1 --image=curlimages/curl --overrides='{"spec":{"nodeName":"node-1"}}' --rm -it --restart=Never -n messaging -- curl -s http://notification-service.messaging.svc.cluster.local:8080/health
```

**Step 7: Check if kube-proxy is syncing**

```bash
kubectl logs -n kube-system -l k8s-app=kube-proxy --tail=100 | grep -i "sync\|error\|service"
```

## Commands

```bash
# Verify endpoints
kubectl get endpoints notification-service -n messaging -o yaml

# Check pod-to-pod connectivity
for ip in 10.244.1.5 10.244.2.8 10.244.3.2; do
  echo "Testing $ip..."
  kubectl exec -it <test-pod> -n messaging -- curl -s --connect-timeout 3 http://$ip:8080/health
done

# Check kube-proxy pods
kubectl get pods -n kube-system -l k8s-app=kube-proxy -o wide

# Check kube-proxy logs
kubectl logs -n kube-system -l k8s-app=kube-proxy --tail=100

# Check kube-proxy configuration
kubectl get cm kube-proxy -n kube-system -o yaml

# Check iptables rules for the service
kubectl debug node/<node> -it --image=busybox -- iptables -t nat -L KUBE-SEP-XXXXXXXXX -v

# Check service selector matches pod labels
kubectl get svc notification-service -n messaging -o jsonpath='{.spec.selector}'
kubectl get pods -n messaging -l app=notification-service --show-labels

# Restart kube-proxy if rules are corrupted
kubectl rollout restart daemonset kube-proxy -n kube-system

# Test Service DNS resolution
kubectl exec -it <test-pod> -n messaging -- nslookup notification-service.messaging.svc.cluster.local

# Check if any endpoint is NOTReady
kubectl get endpoints notification-service -n messaging -o jsonpath='{.subsets[*].addresses[*].conditions.ready}'

# Monitor traffic distribution
kubectl exec -it <test-pod> -n messaging -- bash -c "for i in {1..100}; do curl -s http://notification-service.messaging.svc.cluster.local:8080/health -o /dev/null -w '%{remote_ip}\n'; done" | sort | uniq -c
```

## Root Cause

| Root Cause | Evidence | How to Eliminate |
|---|---|---|
| **kube-proxy Rules Corrupted** | iptables rules only show 1 endpoint | Restart kube-proxy daemonset |
| **kube-proxy Not Running** | kube-proxy pod CrashLoopBackOff | Fix kube-proxy, check config |
| **Node Maintenance Disrupted Rules** | Rules on specific nodes are wrong | Restart kube-proxy on affected nodes |
| **IPVS Table Out of Sync** | IPVS mode, rules not updated | Restart kube-proxy, check IPVS |
| **NetworkPolicy Blocking** | Policy allows only specific pod | Update NetworkPolicy |
| **Endpoint NOTReady** | One endpoint shows ready:false | Fix readiness probe |
| **DNS Cache Stale** | Client using cached DNS | Clear DNS cache |

## Immediate Mitigation

```bash
# 1. Restart kube-proxy to resync rules
kubectl rollout restart daemonset kube-proxy -n kube-system

# 2. Verify restart completed
kubectl get pods -n kube-system -l k8s-app=kube-proxy -w

# 3. Test connectivity after restart
kubectl exec -it <test-pod> -n messaging -- curl -s http://notification-service.messaging.svc.cluster.local:8080/health

# 4. If kube-proxy restart doesn't help, check iptables directly
kubectl debug node/<node> -it --image=busybox -- iptables -t nat -L KUBE-SERVICES

# 5. Emergency: use headless service with direct pod IPs
kubectl patch svc notification-service -n messaging --type='json' -p='[
  {"op": "replace", "path": "/spec/clusterIP", "value": "None"}
]'
```

## Permanent Fix

```yaml
# Ensure kube-proxy is healthy and configured correctly
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-proxy
  namespace: kube-system
data:
  config.conf: |
    mode: "ipvs"
    ipvs:
      scheduler: "rr"
      syncPeriod: "30s"
      minSyncPeriod: "2s"
    conntrack:
      maxPerCore: 32768
      tcpCloseWaitTimeout: "1m0s"
      tcpEstablishedTimeout: "5m0s"
```

## Monitoring

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: service-traffic
spec:
  groups:
  - name: service.rules
    rules:
    - alert: ServiceEndpointMismatch
      expr: kube_endpoints_address{namespace="messaging",endpoint="notification-service"} != kube_deployment_status_ready_replicas{namespace="messaging",deployment="notification-service"}
      for: 5m
      labels:
        severity: warning

    - alert: KubeProxyNotRunning
      expr: kube_pod_status_phase{namespace="kube-system",pod=~"kube-proxy.*",phase="Running"} == 0
      for: 5m
      labels:
        severity: critical
```

## Security

- **kube-proxy permissions**: kube-proxy requires root access on nodes — ensure RBAC is minimal
- **IPVS vs iptables**: IPVS is more performant for large-scale services
- **Network policies**: Ensure NetworkPolicies don't accidentally block Service traffic

## Production Considerations

- **IPVS mode**: For large clusters (>1000 services), use IPVS mode for better performance
- **Service mesh**: Consider Istio/Linkerd for advanced load balancing and observability
- **Circuit breaking**: Implement circuit breakers to prevent cascade failures
- **Retry budgets**: Configure retry budgets in service mesh to prevent retry storms

## Senior-Level Answer

"The issue is likely kube-proxy rule corruption after node maintenance. kube-proxy maintains iptables (or IPVS) rules that implement Service load balancing. After maintenance, these rules might not have resynced. I'd verify all 3 endpoints are healthy, then test direct pod connectivity. If all pods respond, I'd check kube-proxy status and logs, then restart it. The fix is restarting kube-proxy daemonset, which forces a full resync of Service rules. For prevention, I'd use IPVS mode (more resilient to rule corruption), set up monitoring for kube-proxy health, and implement proper drain/uncordon procedures during node maintenance."

## Architect-Level Answer

"At the architecture level, kube-proxy-based load balancing is limited — it's L4 only, has no health-aware routing, and can't do traffic splitting. I'd implement: (1) Service mesh (Istio/Linkerd) for L7 load balancing with health-aware routing, (2) Ingress controller for external traffic with advanced routing, (3) kube-proxy in IPVS mode for better performance and reliability, (4) Monitoring of kube-proxy sync status and iptables rule count, and (5) Automated kube-proxy restart on node maintenance. The key architectural principle is that Service load balancing should be health-aware and observability-rich, not just iptables rules."

## Follow-Up Questions

1. "What's the difference between kube-proxy iptables mode and IPVS mode?"
2. "How does kube-proxy handle endpoint updates when pods are added or removed?"
3. "Explain the DNAT (Destination NAT) chain in kube-proxy iptables rules."
4. "How would you debug a Service that resolves via DNS but connections time out?"
5. "What's the difference between a headless Service and a ClusterIP Service for load balancing?"
