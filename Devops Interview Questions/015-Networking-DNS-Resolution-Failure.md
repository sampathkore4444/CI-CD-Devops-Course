# 015. DNS Resolution Failure in Production

## Scenario

A microservices application running on a Kubernetes cluster suddenly cannot resolve service names. Internal DNS resolution is failing intermittently. Some services can reach each other while others cannot. The application uses CoreDNS for internal DNS resolution. The issue started approximately 30 minutes ago after a cluster upgrade was performed. Services that were running before the upgrade can still communicate with each other, but newly created pods after the upgrade are experiencing DNS resolution failures. The CoreDNS pods are running and show no errors in their logs. kubectl exec into a failing pod and running `nslookup kubernetes.default.svc.cluster.local` returns "SERVFAIL" intermittently. CPU and memory on CoreDNS pods appear normal. The cluster has 3 worker nodes and CoreDNS is deployed as a Deployment with 2 replicas.

## Interviewer Question

A microservices application suddenly cannot resolve service names. Internal DNS resolution is failing intermittently. Some services can reach each other, others cannot. The application runs on Kubernetes with CoreDNS. How do you investigate and fix?

## What I Should Think About

- DNS resolution failures in Kubernetes typically point to CoreDNS configuration, network policies, or kube-proxy issues
- The fact that pre-upgrade pods work but new pods do not suggests a configuration change or network policy introduced during the upgrade
- Intermittent SERVFAIL could indicate upstream DNS resolution issues (CoreDNS forwarding to external DNS) or internal service discovery issues
- Need to check CoreDNS configuration (Corefile), network policies applied to CoreDNS, and kube-proxy/DNS service endpoint health
- The kube-dns Service ClusterIP might have changed during the upgrade, and pods may be caching stale DNS entries
- Check if CoreDNS has sufficient replicas and if they are scheduled on different nodes for resilience
- Verify that the DNS Service (kube-dns) endpoints are correctly populated and that pods can reach the DNS service IP
- Check for any ResourceQuota or LimitRange changes that might affect CoreDNS pods

## Ideal Answer

Start by confirming the scope of the issue: which pods are affected and which are not. Since pre-upgrade pods work and new pods do not, the most likely cause is a change in the DNS configuration or network connectivity to CoreDNS during the upgrade.

First, check the CoreDNS Corefile configuration. The upgrade may have modified the Corefile, introducing a syntax error or incorrect forward rules. Run `kubectl -n kube-system get configmap coredns -o yaml` to inspect the configuration.

Next, verify that the kube-dns Service is correctly pointing to the CoreDNS pods. Check that the Service ClusterIP is stable and that the Endpoints object has the correct pod IPs.

Then, check if network policies were applied during the upgrade that might be blocking DNS traffic (UDP/TCP port 53) from new pods to CoreDNS.

Also investigate kube-proxy to ensure that DNS service traffic is being correctly forwarded to CoreDNS pods. If kube-proxy is using iptables mode, verify that the DNS service chains are present.

Finally, check CoreDNS logs for any errors during the DNS resolution process, particularly around upstream forwarding or cache misses.

## Architecture

```
┌─────────────────────┐
│   Kubernetes API    │
│   Server           │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐     ┌─────────────────────┐
│   CoreDNS Pod 1     │     │   CoreDNS Pod 2     │
│   (Node 1)          │     │   (Node 2)          │
│   :53 (UDP/TCP)     │     │   :53 (UDP/TCP)     │
└─────────┬───────────┘     └─────────┬───────────┘
          │                           │
          │     kube-dns Service      │
          │     ClusterIP: 10.96.0.10 │
          │                           │
          ▼                           ▼
┌─────────────────────────────────────────────────┐
│                 kube-proxy                       │
│    (iptables/ipvs rules for DNS Service)        │
└─────────────────────┬───────────────────────────┘
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
    ┌──────────┐ ┌──────────┐ ┌──────────┐
    │ Pod A    │ │ Pod B    │ │ Pod C    │
    │ (works)  │ │ (fails)  │ │ (fails)  │
    │ pre-     │ │ post-    │ │ post-    │
    │ upgrade  │ │ upgrade  │ │ upgrade  │
    └──────────┘ └──────────┘ └──────────┘
```

## Investigation

1. Identify which pods are affected: compare pre-upgrade vs. post-upgrade pod DNS behavior
2. Check the CoreDNS Corefile configuration for syntax errors or misconfigurations
3. Verify the kube-dns Service and Endpoints are correctly configured
4. Test DNS resolution from a failing pod using dig/nslookup with detailed output
5. Check kube-proxy logs and iptables/ipvs rules for DNS service forwarding
6. Examine CoreDNS logs for errors, timeouts, or upstream resolution failures
7. Check if network policies are blocking DNS traffic from new pods
8. Verify DNS service ClusterIP is reachable from failing pods using nc/curl
9. Test DNS resolution against each CoreDNS pod directly using pod IP
10. Check CoreDNS resource usage and ensure it is not being throttled by resource limits

## Commands

```bash
# Test DNS resolution from a failing pod
kubectl exec -it <failing-pod> -- nslookup kubernetes.default.svc.cluster.local
kubectl exec -it <failing-pod> -- dig kubernetes.default.svc.cluster.local
kubectl exec -it <failing-pod> -- dig kubernetes.default.svc.cluster.local +search +noall +answer +comments

# Check CoreDNS Corefile configuration
kubectl -n kube-system get configmap coredns -o yaml

# Check kube-dns Service and Endpoints
kubectl -n kube-system get svc kube-dns
kubectl -n kube-system get endpoints kube-dns
kubectl -n kube-system describe svc kube-dns

# Check CoreDNS pods status and logs
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide
kubectl -n kube-system logs -l k8s-app=kube-dns --tail=100
kubectl -n kube-system logs -l k8s-app=kube-dns --tail=100 | grep -i "error\|fail\|timeout\|REFUSED\|SERVFAIL"

# Check if CoreDNS pods are running on different nodes
kubectl -n kube-system get pods -l k8s-app=kube-dns -o jsonpath='{range .items[*]}{.metadata.name} {.spec.nodeName} {.status.podIP}{"\n"}{end}'

# Test DNS resolution against each CoreDNS pod directly
kubectl exec -it <failing-pod> -- dig @10.96.0.10 kubernetes.default.svc.cluster.local
kubectl exec -it <failing-pod> -- dig @<coredns-pod-ip> kubernetes.default.svc.cluster.local

# Check kube-proxy iptables rules for DNS
iptables -t nat -L KUBE-SERVICES | grep kube-dns
iptables -t nat -L KUBE-SEP-<hash> | head -10

# Check kube-proxy logs
kubectl -n kube-system logs -l k8s-app=kube-proxy --tail=50

# Check network policies affecting CoreDNS
kubectl get networkpolicy -A
kubectl describe networkpolicy -n kube-system

# Test network connectivity from failing pod to CoreDNS
kubectl exec -it <failing-pod> -- nc -zv 10.96.0.10 53
kubectl exec -it <failing-pod> -- curl -v http://10.96.0.10:53

# Check CoreDNS resource usage
kubectl -n kube-system top pods -l k8s-app=kube-dns
kubectl -n kube-system get deployment coredns -o jsonpath='{.spec.template.spec.containers[0].resources}'

# Check for DNS configuration in pod
kubectl exec -it <failing-pod> -- cat /etc/resolv.conf

# Check if DNS Service ClusterIP has changed
kubectl -n kube-system get svc kube-dns -o jsonpath='{.spec.clusterIP}'

# Restart CoreDNS to force configuration reload
kubectl -n kube-system rollout restart deployment coredns
kubectl -n kube-system rollout status deployment coredns
```

## Root Cause

- **CoreDNS Corefile misconfiguration**: The cluster upgrade may have overwritten or modified the Corefile with incorrect forward rules, causing DNS resolution failures for service names
- **Network policy blocking DNS**: A new network policy applied during the upgrade may be blocking UDP/TCP port 53 traffic from new pods to CoreDNS
- **kube-proxy iptables corruption**: The upgrade may have left kube-proxy iptables rules in an inconsistent state, causing DNS service traffic to be misrouted
- **CoreDNS endpoint stale**: The kube-dns Service endpoints may not have been updated after the upgrade, causing some DNS requests to reach terminated or unreachable CoreDNS pods
- **DNS ClusterIP change**: If the kube-dns Service was recreated during the upgrade, its ClusterIP may have changed, and new pods may be configured with the new IP while old pods still use the old (now invalid) IP

## Immediate Mitigation

1. Roll back the CoreDNS Corefile to the last known working configuration: `kubectl -n kube-system edit configmap coredns`
2. If network policies are blocking DNS traffic, delete or modify them: `kubectl delete networkpolicy <policy-name> -n <namespace>`
3. Restart CoreDNS deployment to force a fresh configuration load: `kubectl -n kube-system rollout restart deployment coredns`
4. Restart kube-proxy pods to regenerate iptables rules: `kubectl -n kube-system rollout restart daemonset kube-proxy`
5. As a temporary workaround, manually configure failing pods to use the correct DNS server IP

## Permanent Fix

1. Validate CoreDNS configuration before and after cluster upgrades using a diff tool
2. Implement automated DNS health checks that continuously verify service resolution
3. Store the known-good CoreDNS configuration in version control and apply it post-upgrade
4. Implement network policy testing in CI/CD to verify DNS traffic is never blocked
5. Configure CoreDNS with at least 2 replicas on different nodes for redundancy
6. Set up CoreDNS monitoring with metrics and alerts for DNS resolution failures
7. Use PodDisruptionBudget to ensure CoreDNS pods are not all evicted during upgrades

## Monitoring

- Monitor CoreDNS metrics: dns_requests_total, dns_response_rcode_count, dns_request_duration_seconds
- Track SERVFAIL and NXDOMAIN response rates
- Alert on DNS resolution latency exceeding 100ms
- Monitor CoreDNS pod health and restart counts
- Track DNS cache hit rate to detect upstream resolution issues
- Set up synthetic DNS resolution tests from each node
- Monitor kube-proxy sync latency and rule count
- Track network policy changes and their impact on DNS traffic

## Security

- Ensure CoreDNS is not exposed outside the cluster (only accessible via ClusterIP)
- Validate that network policies do not allow external DNS queries from the cluster
- Check for DNS rebinding attacks by enabling DNSSEC validation if applicable
- Verify that CoreDNS is running with the least privileges (non-root, read-only filesystem)
- Ensure DNS traffic is not being intercepted or redirected by malicious pods
- Audit CoreDNS configuration changes through Kubernetes audit logs

## Production Considerations

- **High Availability**: Deploy CoreDNS with at least 2 replicas across different nodes. Use PodDisruptionBudget with minAvailable: 1
- **Scalability**: Monitor CoreDNS CPU and memory usage. Scale replicas based on DNS query volume. Consider NodeLocal DNSCache for large clusters
- **Reliability**: Implement DNS retry logic in applications. Use connection timeouts for DNS lookups. Cache DNS results with appropriate TTL
- **Cost**: DNS failures can cascade to all services, causing massive revenue impact. Invest in CoreDNS monitoring and alerting
- **Compliance**: Log DNS queries for security auditing (if required by compliance standards). Ensure DNS resolution is compliant with data residency requirements
- **Operational**: Create a DNS troubleshooting runbook. Test DNS resolution as part of cluster upgrade validation. Document the DNS architecture and failure modes

## Senior-Level Answer

I would first identify the scope by testing DNS resolution from pre-upgrade vs. post-upgrade pods. Then I would check the CoreDNS Corefile for configuration changes introduced during the upgrade, verify the kube-dns Service endpoints are correct, and test DNS resolution against each CoreDNS pod individually. The most likely cause is either a Corefile misconfiguration or a network policy blocking DNS traffic for new pods. I would restore the last known-good Corefile configuration and restart CoreDNS. For prevention, I would implement DNS health checks in the cluster upgrade pipeline and use NodeLocal DNSCache to reduce DNS latency and improve resilience.

## Architect-Level Answer

DNS resolution failures in Kubernetes are a systemic risk because they cascade to all services simultaneously. I would implement a defense-in-depth approach: NodeLocal DNSCache on every node to reduce CoreDNS dependency, CoreDNS horizontal pod autoscaling based on query volume, automated DNS health checks that run during cluster upgrades with automatic rollback on failure, and a centralized DNS monitoring dashboard with SLO-based alerts. I would also establish DNS resolution as a hard gate in the cluster upgrade pipeline, ensuring no upgrade proceeds if DNS health checks fail. For multi-cluster environments, I would implement cross-cluster DNS federation with fallback resolution paths.

## Follow-Up Questions

1. How would you troubleshoot DNS resolution failures for external domains (e.g., api.google.com) versus internal Kubernetes service names?
2. What is the difference between CoreDNS using iptables mode vs. IPVS mode for the kube-dns Service, and how would each affect DNS resolution failures?
3. How would you implement NodeLocal DNSCache and what problem does it solve in large clusters with high DNS query volume?
4. If the CoreDNS PodDisruptionBudget prevented all replicas from being evicted during a node drain, how would you handle the situation?
5. How would you design a DNS disaster recovery strategy for a multi-region Kubernetes deployment where CoreDNS failure in one region should not affect other regions?
