# 018. Network Latency Spike Between Microservices

## Scenario

API response times for a critical microservice jumped from 50ms to 3 seconds. All services are running in the same Kubernetes cluster which spans 3 availability zones. CPU and memory metrics on all pods appear normal. Application logs show no errors or exceptions. Network metrics from the monitoring stack show increased latency specifically between certain pods, particularly between pods running in different availability zones. The latency issue started 2 hours ago and does not correlate with any deployment or configuration change. There were no node or pod restarts. Traffic volume is normal. Both the clients and the services are reporting the slowdown. The affected microservice is the order processing service, which communicates with the inventory and payment services.

## Interviewer Question

API response times jumped from 50ms to 3 seconds. All services are in the same Kubernetes cluster. CPU and memory are normal. Network metrics show increased latency between specific pods. How do you diagnose the root cause?

## What I Should Think About

- A 60x latency increase with normal CPU/memory suggests a network-level issue rather than a compute issue
- Latency between specific pods, especially cross-AZ, points to network path issues: EC2 network interfaces, ENI limits, VPC routing, or overlay network bottlenecks
- Could be a CNI plugin issue causing encapsulation overhead or interrupt storms
- Could be an underlying EC2 instance network bandwidth throttling (EBS-only networking, per-instance network caps) - trap when checking "CPU and memory are normal"
- Could be a noisy neighbor issue on shared underlying hardware
- Could be a DNS issue causing latency (every request triggers DNS resolution with TTL 0)
- Could be TCP retransmissions caused by packet drops on the overlay network
- Could be a datacenter/ENI-level failure where one node has high packet loss
- Check the ENI (Elastic Network Interface) attachment of the affected pods - maybe one node is using a secondary ENI with reduced bandwidth

## Ideal Answer

The key observation is that CPU and memory are normal but network latency between specific pods is high. This points to a network-path issue, not an application-level bottleneck.

My troubleshooting approach:

1. First, determine if the latency is between all pods or only between pods in different AZs. If cross-AZ only, check the VPC for routing issues, or check for a shared network device (like AWS Transit Gateway) that might be saturated
2. Check the CNI plugin (Calico, Cilium, Flannel) for packet loss or encapsulation overhead. On AWS, VPC native CNI (aws-node) avoids encapsulation but uses ENIs per pod
3. Check for EC2 instance network bandwidth throttling on the nodes hosting critical pods. The instance type determines bandwidth; if you have a burstable (t2/t3/t4g) instance, it has a burst balance that may be exhausted
4. Run `ping` or `mtr` from affected pods to identify the exact hop/loss point
5. Check if the issue is TCP retransmission: capture with tcpdump and look for duplicate ACKs, retransmitted SYN/SYN-ACK, or window updates
6. Check the ENI / VPC flow logs for dropped or rejected traffic
7. Check for MTU mismatches: if pod1 and pod2 use different MTU, fragmentation causes high latency
8. Check if one node is running many pods, causing bandwidth contention on the shared NIC

Also verify that the latency correlates with DNS time. If the service uses DNS-based discovery (Kubernetes service names) rather than direct pod IP, each query may trigger a DNS lookup. A slow DNS response (maybe because CoreDNS is affected) can add significant latency.

## Architecture

```
Client → ALB → Service A (Node A - AZ1) ────────────── Service B (Node B - AZ1)
                                        │  same-AZ:       fast (50ms)
                                        │
                                        └────────────── Service C (Node C - AZ2)
                                           cross-AZ:      slow (3s)

Node A (AZ1)          Node C (AZ2)
┌──────────────┐      ┌──────────────┐
│ Service A    │      │ Service C    │
│ eth0 (ENI)   │      │ eth0 (ENI)   │
│ cni0 bridge  │      │ cni0 bridge  │
└──────┬───────┘      └──────┬───────┘
       │                     │
       └────────VPC──────────┘
          route tables, ACLs,
          subnet routing, ENI
```

## Investigation

1. Verify the latency is consistent and not a transient spike: sample it over time
2. Identify which specific pods/pairs show high latency
3. Run echo tests (ping, mtr, traceroute) from affected pod to unaffected pod and to external services
4. Check the Kubernetes network path: does the latency correlate with pod-to-pod vs. pod-to-service?
5. Check VPC Flow Logs for dropped packets between the affected nodes
6. Check TCP zero windows and retransmissions with tcpdump
7. Examine the node's network bandwidth metrics and ENA (Enhanced Networking) stats
8. Check if any node has a degraded ENI or instance network health
9. Check the CNI plugin configuration for MTU, encapsulation, tunnel setup
10. Verify DNS latency is not contributing
11. Check if the issue correlates with specific request types (post vs. get, large payloads)

## Commands

```bash
# Identify the affected pods and their nodes/AZs
kubectl get pods -o wide | grep <service>
kubectl get nodes -o wide --show-labels

# Get node AZs
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name} {"\t"} {.metadata.labels["topology.kubernetes.io/zone"]}{"\n"}{end}'

# Measure latency between pods with ping (ICMP may work if CNI permits)
kubectl exec -it <pod-a> -- ping -c 5 <pod-b-ip>

# Better: use MTR/traceroute for path analysis
kubectl exec -it <pod-a> -- mtr -r -c 10 <pod-b-ip>
kubectl exec -it <pod-a> -- traceroute <pod-b-ip>

# Compare same-AZ vs cross-AZ latency
kubectl exec -it <pod-a> -- curl -o /dev/null -s -w "same-az: %{time_total}s\n" http://<pod-b-ip>:<port>/health
kubectl exec -it <pod-a> -- curl -o /dev/null -s -w "cross-az: %{time_total}s\n" http://<pod-c-ip>:<port>/health

# Capture packets and check for retransmissions
kubectl exec -it <pod-a> -- tcpdump -i eth0 -w /tmp/latency.pcap port <port>
# On the receiving pod
kubectl exec -it <pod-b> -- tcpdump -i eth0 -w /tmp/latency.pcap port <port>

# Analyze the pcap file (copy to local, then use tcpdump or wireshark)
tcpdump -r latency.pcap | grep -i "retransmission\|dup ack\|out-of-order"

# Check node network statistics
netstat -s
ip -s link
# Check for retransmissions, errors, drops:
netstat -s | grep -i "retransmit\|drop\|errors"

# Check ENA (Enhanced Networking) / network interface stats on nodes
ethtool -S eth0 | head -40
ethtool -S eth0 | grep -i "drop\|error\|miss\|fail"

# Check for bandwidth throttling on EC2 (burst credits on t-series)
# From the AWS CLI or by checking
cat /sys/class/net/eth0/statistics/rx_bytes   # (snapshot over time)

# Check CPU steal time (noisy neighbor indicator)
kubectl exec -it <pod-a> -- cat /proc/stat | grep cpu

# Check ENI flow logs
# In VPC flow logs, look for:
# 1 DENY - VPC Network ACL / SG dropping
# 2 REJECT - dropped by route table
# Flow log format: version account-id interface-id srcaddr dstaddr srcport dstport protocol packets bytes start end action log-status

# Query VPC flow logs via Athena or CloudWatch Logs Insights

# Check if the cluster is using ENI-based CNI (aws-node) or Calico overlay
kubectl get ds -n kube-system aws-node -o wide
kubectl get ds -n kube-system calico-node -o wide  # if present

# Check CNI MTU settings
kubectl -n kube-system get ds aws-node -o jsonpath='{.spec.template.spec.containers[*].env}' | jq . | grep -i MTU
# Check docker/containerd networking MTU on nodes
ip link show | grep -i mtu

# Check AWS instance network performance
amazon-linux-extras list | grep -i network
# Or use AWS console: EC2 > Instances > Networking tab
```

## Root Cause

- **One node's ENI degraded or Network bandwidth exhausted**: The node hosting a critical pod has exhausted its network bandwidth (EBS-only, burstable, or a shared NIC limit). ENA (Enhanced Networking) may have degraded
- **Overlay network encapsulation overhead**: CNI overlay (Calico VXLAN/IPIP, Flannel VXLAN) can cause packet loss or fragmentation when MTU is misconfigured, particularly when payloads exceed the reduced MTU
- **Route table / NAT gateway contention**: The VPC route table may be sending a portion of traffic to an overloaded NAT gateway or a transit gateway
- **Noisy neighbor**: Another pod or instance on the underlying host saturating the shared network path
- **TCP retransmission storm**: Packet loss or reordering caused by the underlying overlay causing many retransmissions, inflating latency
- **DNS latency**: CoreDNS or the service's DNS resolution is slow on some requests (e.g., when used for discovery, TTL expiring mid-path)
- **Node Maintenance or degradation**: The node might be undergoing maintenance or health issues causing its ENI path to degrade

## Immediate Mitigation

1. If a specific node is affected, drain it and reschedule pods to healthy nodes: `kubectl drain <node>` then `kubectl uncordon <node>`
2. If the node network itself is degraded, restart the node after draining (with approval)
3. If DNS latency is a factor, increase DNS TTL for service discovery, or switch to direct pod IP / headless service
4. If ENI bandwidth is throttled, temporarily increase the instance type or add another ENI to improve bandwidth
5. If the issue is cross-AZ only, consider re-balancing pods so they reside in the same AZ

## Permanent Fix

1. Match the CNI MTU precisely with the VPC MTU (typically 9001 for jumbo frames, but verify)
2. Use VPC-native CNI (aws-node) rather than overlay to reduce encapsulation overhead, or implement EBPF-based Cilium
3. Enable AWS Enhanced Networking (ENA) on all nodes (if not already)
4. Implement topologically-aware routing (topologySpreadConstraints) in Kubernetes to co-locate communicating pods
5. Add network-level monitoring with latency histograms per pod pair and alert on degradation
6. Tune TCP: increase initial congestion window (initcwnd), enable TCP BBR congestion control on nodes

## Monitoring

- Track network latency between pod pairs (histograms: p50, p95, p99)
- Monitor retransmission and retry rates per node
- Track packet drops at ENI/veth interfaces
- Monitor DNS resolution time for service discovery
- Alert on cross-AZ latency exceeding a threshold (e.g., 100ms)
- Track AWS VPC Flow Logs for DENY/REJECT patterns
- Monitor TCP zero-window events and out-of-order packets

## Security

- Ensure cross-AZ network paths do not accidentally traverse public or transit networks without encryption
- Use network policy to restrict which pods can reach which services
- Verify that the network tooling you run (tcpdump, mtr) is restricted to debug pods and containers
- Review route tables to ensure no traffic is unexpectedly routed through public or shared infrastructure
- Ensure flow log data is retained in S3 for compliance and audit purposes

## Production Considerations

- **High Availability**: Design services to be AZ-agnostic with anti-affinity and topology spread constraints to avoid cross-AZ latency being critical
- **Scalability**: Choose EC2 instance types with sufficient network bandwidth for the pod density. Use placement groups for cluster-internal high-throughput traffic
- **Reliability**: Add retry and circuit breaker patterns to tolerate elevated latency. Implement client-side timeouts that align with observed latency SLOs
- **Cost**: Network bandwidth is a cost factor on EC2/ENI. Using topology-aware scheduling reduces cross-AZ traffic and costs
- **Compliance**: Maintain flow log data and latency metrics for audit. Ensure cross-AZ traffic encryption is used if required
- **Operational**: Create a runbook for network latency incidents. Document the CNI architecture and expected network metrics for each node type

## Senior-Level Answer

Since CPU and memory are normal, this is a network-path issue. I'd identify the exact pod pair with high latency, compare same-AZ vs cross-AZ latency, and run mtr/tcpdump between them. I'd check for retransmissions, ENI bandwidth throttling, MTU mismatches, and DNS contribution. The fix would depend on findings: tune MTU/CNI, rebalance pods across AZs, or scale/upgrade the affected node's ENI.

## Architect-Level Answer

Latency spikes with healthy CPU/memory are a networking architecture problem. The long-term fix is to move to a service mesh (e.g., Istio) with per-hop latency tracing and L7 metrics, giving us visibility into inter-service paths. I would also standardize the CNI on Cilium with eBPF for efficient packet processing, enable topology-aware routing so communicating pods prefer same-AZ nodes, and set up network-layer SLOs with alerting. Crucially, I would design the deployment with a tolerance for cross-AZ failure: if an entire AZ degrades, the cluster should still serve critical traffic. This means running critical services in multiple AZs and testing failover under injection.

## Follow-Up Questions

1. How would you distinguish between application-level latency (e.g., slow serialization, DB calls) and network-level latency from the metrics alone?
2. What are the trade-offs of co-locating communicating pods in the same AZ to reduce latency, against the reliability of spreading them across AZs?
3. How would you identify a "noisy neighbor" issue on a shared EC2 host without access to the hypervisor?
4. If VPC Flow Logs show all traffic as "ACCEPT" yet latency is high, what does that tell you about where you should look next?
5. How would you design a network latency SLO and the alerting that enforces it for a multi-AZ microservices platform?