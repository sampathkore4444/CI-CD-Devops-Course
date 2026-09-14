# 017. TCP Connection Timeout Between Services

## Scenario

Service A is intermittently timing out when connecting to Service B on port 8080. Both services are deployed in the same Kubernetes cluster. Service A is a Node.js API service, and Service B is a Java backend service. `tcpdump` captured on the Service B pod shows SYN packets arriving, but no SYN-ACK is being sent in reply. The issue happens roughly 4% of the time and appears to be random. Service A's connection timeout is set to 3 seconds, after which it marks the connection as failed and retries. The retry often succeeds, which explains why only a small percentage of overall requests fail. The cluster uses a CNI (Container Network Interface) plugin for pod networking. Nodes are running on AWS EC2 instances. The issue has been ongoing for a week and has not responded to pod restarts or scaling changes.

## Interviewer Question

Service A is intermittently timing out when connecting to Service B on port 8080. Both services are in the same Kubernetes cluster. tcpdump shows SYN packets being sent but no SYN-ACK. How do you determine if it's a firewall, network policy, or application issue?

## What I Should Think About

- SYN packets arriving at Service B but no SYN-ACK sent means the packets are reaching the pod, so it's not purely a network routing issue upstream of Service B
- If the SYN arrives but no SYN-ACK, the issue is either: iptables rules on the node dropping SYN, a network policy or CNI plugin dropping the packet, a full connection table/backlog, or the application's accept queue being full
- The kernel's SYN backlog (tcp_max_syn_backlog) can overflow if the application is not accepting connections fast enough
- Need to check the CNI plugin (Calico, Flannel, Cilium, etc.) for policy rules or IP tables issues
- Check for network policy that might be rate-limiting or dropping SYN packets
- Check the node-level iptables rules and conntrack table for connection tracking issues
- If the packet reaches the pod network namespace, it should be answered; verify whether the SYN arrives in the pod itself with tcpdump on the pod veth interface
- Check for a possible IPv6 vs IPv4 mismatch or TCP segmentation offload issues
- Check the number of connections in SYN_RECV state on Service B pod which would indicate backlog overflow
- Perhaps the issue is conntrack table exhaustion at the node level causing SYN drops

## Ideal Answer

This is a classic TCP connection establishment issue where SYN arrives but is not answered. The diagnosis hinges on isolating where the SYN is dropped. Since you see SYN at the pod, I would first confirm the SYN reaches the pod's network namespace and then check for firewall or CNI plugin interference.

The packet path is: Service A pod -> Node 1 veth -> bridge/CNI -> Overlay network -> Node 2 routing -> veth for Service B pod -> iptables -> Service B.

Key troubleshooting steps:
1. Run tcpdump on Service B pod to confirm whether SYN-ACK is generated locally. If not, check the pod's own iptables
2. Check the conntrack table size and status on the nodes: if the table is full, new connections may be dropped
3. Check if a NetworkPolicy or Calico policy is blocking or rate-limiting SYN packets
4. Check kernel parameters: tcp_max_syn_backlog, net.core.somaxconn, and net.ipv4.ip_local_port_range
5. Check the application's accept backlog and connection handling: if the app is single-threaded or its accept queue is full, the kernel will not SYN-ACK
6. Check if there is a TCP offload or MTU issue causing some SYNs to be dropped at the network driver level

## Architecture

```
Service A Pod                    Node 1                    Node 2                 Service B Pod
┌────────────┐              ┌──────────────┐          ┌──────────────┐          ┌────────────┐
│ APP        │              │  veth A      │          │  veth B      │          │  ET0       │
│ Node.js    │┐── SYN ─────>│  cni0       │         >│  eth0        │───SYN───>│  Java App  │
│            ││             │  eth0       │          │  cni0        │          │  :8080     │
└────────────┘└             └──────────────┘          └──────────────┘          └────────────┘
     │  app                    │  iptables              │  iptables/               │  iptables
     │  socket                 │  (NAT/fwd)             │  conntrack               │  (input)
     ▼                         ▼                        ▼                           ▼
  Queueing                     │                        │
  SYN dropped?                 │                        │                        SYN received
                               │                        │                        but no SYN-ACK
                               ▼                        ▼
                         SYN might be dropped        SYN might be dropped
                         at conntrack table         at CNI policy
                         or forwarded by iptables
```

## Investigation

1. Reproduce the failure and capture packets simultaneously on Service A, Service B, and the nodes
2. Run `tcpdump` on Service B pod to verify whether SYN is even reaching the pod's network namespace
3. Check kernel networking parameters on the Service B node: somaxconn, tcp_max_syn_backlog
4. Check conntrack table usage on both nodes
5. Examine network policies and CNI logs (Calico/Cilium) for dropped packets
6. Check iptables rules on the node and inside the pod
7. Monitor SYN_RECV and SYN_SENT socket counts during the failure window
8. Test connectivity directly from node to pod to eliminate the overlay network
9. Check MTU values across the path for possible packet fragmentation issues
10. Examine application-level connection handling: thread pool, max connections, connection initialization time

## Commands

```bash
# Capture packets on Service B pod network namespace
# First find Service B pod's container ID
kubectl get pod <service-b-pod> -o jsonpath='{.status.containerStatuses[0].containerID}'

# Capture on Service B pod's network namespace (requires running on the same node)
# Find the veth interface name for the pod
kubectl exec -it <service-b-pod> -- cat /proc/net/dev

# Capture on both side simultaneously
# On Service A pod:
kubectl exec -it <service-a-pod> -- tcpdump -i eth0 -w /tmp/syn.pcap port 8080
# On Service B pod:
kubectl exec -it <service-b-pod> -- tcpdump -i eth0 -w /tmp/synack.pcap port 8080

# On the node where Service B runs, check network statistics
netstat -s | grep -i "SYN\|LISTEN\|overflow\|drop"
ss -s
ss -ant | grep 8080
ss -lnt | grep 8080

# Check kernel parameters on the node
sysctl net.core.somaxconn
sysctl net.ipv4.tcp_max_syn_backlog
sysctl net.ipv4.tcp_syncookies
sysctl net.netfilter.nf_conntrack_max
sysctl net.netfilter.nf_conntrack_count

# Check conntrack table usage
sysctl net.netfilter.nf_conntrack_count
cat /proc/sys/net/netfilter/nf_conntrack_max

# Check for conntrack drop events
dmesg | grep -i conntrack
dmesg | grep -i "nf_conntrack"

# Check iptables rules on the node that affect the pod
iptables -L -n -v | grep -i "drop\|reject\|8080"
iptables -t nat -L -n -v | grep 8080

# Test connection directly from the node to the pod IP
# This bypasses the overlay network
curl -v --connect-timeout 3 http://<service-b-pod-ip>:8080/health

# Test from another pod on the same node
kubectl exec -it <other-pod> -- curl -v --connect-timeout 3 http://<service-b-pod-ip>:8080/health

# Check for connection tracking issues by restarting conntrack (if using iptables)
# Caution: this resets all connection state on the node

# Check CNI plugin logs (Calico)
kubectl -n kube-system logs -l k8s-app=calico-node --tail=20
kubectl -n kube-system get ds calico-node -o wide

# Check calico IPSet or Felix logs
kubectl -n kube-system get pod -l k8s-app=calico-node -o name | xargs -I{} kubectl -n kube-system logs {} --tail=50

# Check network policies
kubectl get networkpolicy -A -o wide

# Test for MTU issues
kubectl exec -it <service-a-pod> -- ping -M do -s 1400 <service-b-pod-ip>
kubectl exec -it <service-a-pod> -- ping -M do -s 1500 <service-b-pod-ip>

# Check IF the SYN-ACK is generated but dropped on the way back:
# Capture on the node hosting Service B's veth interface
tcpdump -i <veth-interface> 'tcp[tcpflags] & (tcp-syn|tcp-ack) != 0 and port 8080'

# Check SYN_RECV count during failure period
watch -n 1 'ss -ant | grep -E "SYN_RECV|SYN_SENT" | wc -l'
```

## Root Cause

- **Conntrack table exhaustion**: The node's connection tracking table is full, causing new connections to be dropped silently. SYNs arrive but cannot be tracked, so no SYN-ACK is sent
- **Application backlog overflow**: The application's accept queue lacks space, so the kernel stops responding to new connection requests. The backlog is full because the application is processing requests slower than new connections arrive
- **Network policy (CNI) drop**: A Calico/Cilium NetworkPolicy is configured to drop a percentage of SYN packets (for rate limiting) or is intermittently applying rules
- **iptables drop rules**: A firewall or CNI-added iptables rule is intermittently dropping SYN packets from Service A
- **Node conntrack parameter misconfiguration**: net.netfilter.nf_conntrack_max is too low for the node's traffic load. When exceeded, new connections are dropped

## Immediate Mitigation

1. If conntrack table is full, temporarily raise nf_conntrack_max: `sysctl -w net.netfilter.nf_conntrack_max=65536`
2. Increase kernel backlog parameters temporarily: `sysctl -w net.core.somaxconn=65535` (requires application to use SO_REUSEADDR and call listen() with a larger backlog)
3. Restart Service B pod to clear stale connection states: `kubectl delete pod <service-b-pod>` (if a single pod restart resolves it, that points to the application backlog)
4. Temporarily disable CNI policy enforcement to isolate the policy as the cause (use caution in production)
5. Use Retry-once logic in Service A: it often succeeds on retry, so the immediate availability concern is reduced

## Permanent Fix

1. Set proper values for conntrack, backlog, and socket parameters on all nodes at boot (sysctl.conf)
2. Configure the application to use a suitable listen backlog matching its concurrency model
3. Tune the Kubernetes CNI/network policy to not interfere with connection establishment
4. Monitor conntrack table usage and alert before it approaches the max limit
5. Consider switching to a CNI that handles connection tracking more efficiently (Cilium's eBPF handles NAT in kernel space)
6. Implement proper PodDisruptionBudgets and restart strategies to blast radius of the fixes

## Monitoring

- Track conntrack table usage: net.netfilter.nf_conntrack_count percent utilization
- Monitor SYN_RECV and SYN_SENT socket counts per pod
- Track SYN drop events from kernel logs
- Monitor TCP connection establishment latency (handshake time) between pods
- Alert on conntrack table utilization exceeding 80%
- Monitor network policy enforcement metrics from CNI (num policies, rejected packets)
- Track socket backlog overflow events in kernel: netstat -s "times the listen queue of a socket overflowed"

## Security

- Ensure the CNI policy provides proper network segmentation between services (zero trust)
- Verify iptables rules enforce least-privilege access between namespaces
- Ensure connection tracking is not being abused as an amplification vector (e.g., SYN flood)
- Keep kernel parameters in line with node security hardening baselines
- Check that internal services do not expose debugging ports or metrics to unauthorized pods

## Production Considerations

- **High Availability**: Ensure Service A and B are deployed on different nodes with anti-affinity rules to avoid single-node failures causing total service loss
- **Scalability**: Understand the connection rate your applications generate. Size nodes and conntrack tables accordingly. Use horizontal pod autoscaling
- **Reliability**: Implement connection retries with exponential backoff in Service A. Use circuit breakers to fail fast
- **Cost**: Connection-related outages are typically resolved more quickly with monitoring in place. Preventing these failures has a higher ROI than adding more compute
- **Compliance**: Log network-level rejections for audit. Ensure security policies are reviewed periodically
- **Operational**: Document the TCP connection path and the failure modes in an architecture diagram. Maintain a runbook for socket/connection issues

## Senior-Level Answer

The fact that SYN arrives at Service B but no SYN-ACK is sent firmly eliminates the overlay network and points to either the kernel's socket accept queue being full, a CNI policy dropping SYN, or conntrack table exhaustion. I would run tcpdump on both the pod veth interface and inside the pod, check `ss -lnt | grep 8080` for the backlog, check conntrack usage with `sysctl net.netfilter.nf_conntrack_*`, and check Calico/Cilium policy logs. The remediation depends on which check fails: increase conntrack max, tune somaxconn, or adjust the network policy.

## Architect-Level Answer

Intermittent SYN drops like this are a symptom of untested capacity boundaries in the network stack. I would introduce eBPF-based networking (Cilium) which offloads connection tracking to the kernel XDP path and dramatically reduces SYN-to-SYN-ACK latency at scale. I would also standardize kernel tuning parameters across all nodes as part of the cluster baseline (conntrack, somaxconn, backlog sizes) and enforce them via infrastructure-as-code. Finally, I would add application-level connection pooling and health checking so Service A only establishes connections to healthy Service B instances, and implement circuit breaker patterns to isolate failures.

## Follow-Up Questions

1. How would your diagnosis change if the SYN-ACK is generated by Service B but dropped on the return path to Service A?
2. What is the difference between a missing SYN-ACK and a RST being sent in response to SYN, and what does each indicate?
3. How would you use eBPF or Cilium's Hubble to trace the exact packet path with causality for these intermittent drops?
4. If the issue was load-balancer-related (e.g., IPVS/iptables rewriting), how would that differ from a pod-level issue?
5. How would you distinguish between a kernel-level backlog overflow and an application-level connection handling issue in the Node.js service on Service A?