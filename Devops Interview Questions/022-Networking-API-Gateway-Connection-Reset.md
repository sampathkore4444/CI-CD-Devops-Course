# 022. API Gateway Returning Connection Reset Errors

## Scenario

Clients are receiving TCP RST (connection reset) errors from an API gateway. About 2% of all requests fail. The gateway (Kong running in Docker) shows no errors in its logs. Backend services are healthy with normal response time. The load balancer (AWS NLB in front of Kong) shows normal metrics — no spike in connection errors or rejected connections. The API gateway is deployed across multiple EC2 instances (3 nodes) in an autoscaling group behind the NLB. The clients are external (partner APIs over the internet) connecting via TLS. The issue started 2 days ago after a deployment of the Kong gateway that increased the number of worker processes from 1 to 4 per node. Around 2% of requests are failing with certain clients seeing "connection reset by peer" while others don't.

## Interviewer Question

Clients are receiving TCP RST (connection reset) errors from an API gateway. About 2% of all requests fail. The gateway shows no errors in logs. Backend services are healthy. Load balancer shows normal metrics. How do you investigate this at the network level?

## What I Should Think About

- A TCP RST (connection reset) means a network device or a host actively reset the connection — this is different from a timeout (silent drop)
- The RST can originate from: the client, the NLB, Kong, a WAF/IDS/firewall inline (like AWS Network Firewall), or an intermediate device
- The application says "no errors in logs" — but Kong/Docker may not log TCP-level resets; the reset is happening below the application (TCP/L4-L3) or at the TLS layer
- A change (worker processes from 1 to 4) is the leading suspect: the deployment could have introduced a connection handling issue (e.g., each worker handling a subset of connections, or a race condition at the worker level)
- Check the NLB for "resets" metric specifically (`AWS/NetworkELB` has a `TCP_Reset_Count` metric). NLB shows normal metrics in the UI maybe, but this specific metric might be missed
- Common causes of TCP RST at the gateway:
  - Kernel `tcp_abort_on_overflow` set, so when the accept queue overflows it sends RST
  - The app's connection/request size limits per worker hitting a cap and dropping connections
  - Idle/keepalive handling: if Kong closes idle keepalive connections aggressively while the client is mid-request, a RST can result
  - An inline security/firewall device (WAF, IDS) blocking a portion of traffic — e.g., rate limiting or signature rules catching some requests
  - TCP retransmission after a partial connection establishment (SYN/SYN-ACK issue) causing a reset
  - NLB target connection draining / connection timeout settings
  - MySQL/Redis connection pool exhaustion in Kong causing requests to fail and the connection to be reset
- Need to capture the actual RST: tcpdump on a Kong node and see whether the RST is sent from Kong or received from upstream, and correlate with request patterns (client IP, TLS SNI, path, method)

## Ideal Answer

The key facts: RST (not timeout), 2% failure rate, backend healthy, LB shows normal metrics, and the change (workers 1→4). I would treat this as a network-level connection lifecycle issue.

My approach:
1. Determine the RST source. Capture on the Kong node: `tcpdump` for the client's traffic. Look at whether Kong sends the RST (e.g., RST in the reply from Kong IP to client) or the client sends RST
2. Check the NLB's `TCP_Reset_Count` metric — a high value points to resets happening at the NLB, which can be caused by target health issues, connection draining, or client behavior. The UI "normal metrics" may have missed this one
3. Inspect Kong's connection handling: check the number of open connections per worker, TCP accept queue overflow, SYN queue overflow
4. Check kernel parameters: `tcp_abort_on_overflow` and `net.core.somaxconn`. If `tcp_abort_on_overflow=1`, an overflowed accept queue sends RST instead of silently dropping the SYN
5. Look at keepalive/idle timeout handling: if the client or an intermediate device sends a request on a keepalive connection that Kong has closed (idle timeout), a RST is sent/expected
6. Check if the WAF/IDS/firewall inline is resetting a subset of connections (e.g., matched patterns, or rate-limit threshold)
7. Check if the issue correlates with a specific TLS SNI / API key / source IP range (certain partners more affected)

If I find the RST originates from Kong's kernel, the likely cause is the accept-queue overflow due to the increased worker count oversubscribing file descriptors or the shared accept queue, and I'd fix the backlog/somaxconn and revisit the worker count.

## Architecture

```
Partner clients (internet)
        │  TLS
        ▼
┌──────────────────────┐
│  AWS NLB             │
│  (TCP listeners)     │
│  TCP_Reset_Count ?   │
└──────────┬───────────┘
           │
┌──────────┴───────────┐
│  (optional inline    │
│   WAF/IDS/firewall)  │  ← could send RST
└──────────┬───────────┘
           ▼
┌──────────────────────┐
│  Kong API Gateway    │
│  (Docker on EC2)     │
│  worker processes ×4 │
│  file descriptors    │
│  backlog/somaxconn   │
│  keepalive/idle      │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│  Backend services    │
│  (healthy, latency ok)│
└──────────────────────┘
```

## Investigation

1. Capture packets on the Kong node to attribute the RST source
2. Check NLB `TCP_Reset_Count` metric for the target group
3. Check Kong open connections, accept queue overflow, SYN queue overflow
4. Review kernel TCP parameters (tcp_abort_on_overflow, somaxconn, tcp_max_syn_backlog)
5. Check Kong logs and access logs at the right verbosity for connection-related events
6. Check the client profile: same clients failing repeatedly or random?
7. Check the relationship to the worker-process change (bisect by reverting to 1 worker and observing)
8. Check for inline WAF/IDS/third-party network appliance in the path
9. Check file descriptor / connection limits on the Kong host vs. number of open sockets
10. Check the NLB connection draining and target registration health during the failure window

## Commands

```bash
# Capture on the Kong node to see RST source
# On a Kong node:
sudo tcpdump -i eth0 -nn 'tcp[tcpflags] & (tcp-rst) != 0' -c 20

# Full capture of one failing connection (correlate by client IP)
sudo tcpdump -i eth0 -nn -s 0 -w /tmp/kong_rst.pcap host <client-ip> and port 443

# Analyze the capture: which side sends the RST?
tcpdump -r /tmp/kong_rst.pcap -nn 'tcp[tcpflags] & (tcp-rst) != 0'
# RST with SRC = Kong IP ⇒ Kong resets
# RST with SRC = client IP ⇒ client resets (may be triggered by something upstream)

# Check NLB TCP reset count metric
aws cloudwatch get-metric-statistics \
  --namespace AWS/NetworkELB \
  --metric-name TCP_Reset_Count \
  --dimensions Name=LoadBalancer,Value=<nlb-name> \
  --start-time <start> --end-time <end> --period 60 --statistics Sum \
  --region <region>

# Check accepted/registered/healthy target count
aws elbv2 describe-target-health --target-group-arn <tg-arn> --region <region>

# On Kong node: check open sockets and connections
ss -ant | grep ESTABLISHED | wc -l
ss -ant | grep SYN_RECV | wc -l
ss -ant | grep CLOSE_WAIT | wc -l
ss -lnt | head -20

# Check accept queue usage for the Kong listen port
ss -lnt | awk '{print $2}' | sort | uniq -c

# Check if listen backlog/accept queue is overflowing (kernel counters)
netstat -s | grep -i "overflow"
netstat -s | grep -i "resets"
netstat -s | grep -i "connections"

# Check kernel TCP tuning parameters
sysctl net.ipv4.tcp_abort_on_overflow
sysctl net.core.somaxconn
sysctl net.ipv4.tcp_max_syn_backlog
sysctl net.ipv4.ip_local_port_range

# If tcp_abort_on_overflow=1, that's a strong signal:
# connections are reset when the accept queue overflows

# Check Kong worker connections and proxying
docker ps | grep kong
docker stats --no-stream  | head -10
docker exec -it <kong-container> ps aux | grep -i worker

# Check Kong access logs for the failed requests (with proxy_status)
docker logs <kong-container> --tail=100 | grep -E "502|504|reset"

# Check file descriptor limits on the Kong host
cat /proc/$(pgrep -o nginx)/limits | grep "Max open files"
# (Kong uses nginx under the hood; check the nginx workers)

# Check each nginx worker's socket count
ls /proc/$(pgrep -o nginx)/fd | wc -l

# Test from the Kong node against a backend to isolate gateway vs. upstream
curl -v --connect-timeout 5 http://<backend-service>:<port>/health

# Reproduce on a test client
# loop with keepalive to find the reset rate
for i in $(seq 1 500); do
  curl -s -o /dev/null -w "%{http_code}\n" https://<gateway-host>/path
done | sort | uniq -c

# Check the Docker/Kong network interface / bridge MTU
docker network inspect bridge | grep -i mtu
```

## Root Cause

- **Accept queue (backlog) overflow with tcp_abort_on_overflow=1**: After increasing Kong workers from 1 to 4, the per-worker accept queues and the shared listen backlog overflow under load; with `tcp_abort_on_overflow=1`, the kernel sends RST instead of queueing — explaining the ~2% of failed connections and "no errors" in the app logs (the kernel handles it)
- **Kong keepalive/idle timeout mismatch**: Kong's `keepalive_timeout` (or the upstream keepalive) is shorter than the client's, so Kong closes the connection while the client is mid-request; the client sees a reset
- **File descriptor exhaustion**: The increased worker count multiplied fd usage; when fds are exhausted, new connections fail to accept and get reset
- **Inline WAF/NIDS/network appliance**: A security device in the path rate-limits or blocks a subset of requests (matched patterns), sending RST on the filtered traffic
- **NLB connection draining / idle timeout**: During autoscaling events, the NLB's connection draining (or a short idle timeout) sends resets on in-flight connections
- **TCP retransmission interaction**: After a partial connection establishment, a retransmitted packet eventually gets answered with RST from the peer

## Immediate Mitigation

1. Temporarily set `sysctl -w net.ipv4.tcp_abort_on_overflow=0` on the Kong nodes to stop kernel-level RST resets (if that parameter is the cause)
2. Increase the listen backlog: `sysctl -w net.core.somaxconn=4096` and set the `listen` backlog in Kong's config (`nginx_http 'listen ... backlog=4096;'` or increase `worker_connections`)
3. Revert the worker process count from 4 to the previous 1 (or to 2) to confirm/remove the change as the cause
4. Increase file descriptor limits (`ulimit -n`) and Kong's `worker_rlimit_nofile`/`worker_connections`
5. Check/align keepalive timeouts between Kong and clients (or between NLB/Kong)
6. If an inline WAF is the cause, adjust its rules or temporarily bypass it with approval

## Permanent Fix

1. Right-size Kong worker count to the host's CPU/fd capacity rather than arbitrarily increasing it
2. Increase TCP tuning on all gateway hosts via boot config: somaxconn, tcp_max_syn_backlog, tcp_abort_on_overflow=0, and set proper listen backlog in Kong config
3. Align keepalive/idle timeouts across the chain: NLB idle timeout ≥ Kong upstream keepalive ≥ client idle
4. Use a dedicated monitoring for connection resets: alert on TCP_Reset_Count spike on the NLB
5. Standardize Kong configuration with correct worker settings via IaC and validate with load testing before deploy
6. Add connection metrics (accept queue length, reset counts) to Kong monitoring

## Monitoring

- Alert on NLB `TCP_Reset_Count` spikes (per 1-minute window)
- Monitor kernel TCP counters (resets, overflow) per gateway host
- Track Kong worker connection count vs. fd limit
- Monitor accept queue length and SYN queue overflow on gateway hosts
- Track client-visible error rate (RST) per endpoint / partner
- Monitor WAF/IDS inline device dropped/reset counters
- Set up alerts on connection rate anomaly (2-5% reset rate) 

## Security

- TCP resets can also be caused by an IDS/IPS or inline security appliance that blocks matched patterns; review its rules for the affected traffic
- If an inline WAF resets connections, ensure it's not dropping legitimate API calls (tune rules based on the traffic pattern)
- Ensure TLS is properly terminated and the reset is not masking a TLS handshake issue (check the SNI/host header during resets)
- Protect the gateway from SYN-flood style attacks (which can exhaust the accept queue) — enable SYN cookies if needed
- Keep the gateway's connection limit per client bounded to prevent one partner from exhausting connections

## Production Considerations

- **High Availability**: Run Kong across multiple nodes/AZs behind the NLB so a single gateway node's accept queue overflow doesn't take out the API. The NLB's connection draining should complete cleanly during scale events
- **Scalability**: Size worker count, fd limits, and kernel queues to the peak connection rate. Use connection pooling from clients to the gateway. Monitor and autoscale based on connection rate plus CPU
- **Reliability**: The reset pattern is a reliability risk even at 2%. Use overlapping timeouts (client < gateway < backend pattern) so no hop resets an in-flight request. Define and measure an "API availability" SLO that includes connection resets
- **Cost**: Frequent RST-based failures need more support load and can drive churn. Investing in proper kernel tuning and monitoring is cheap compared to partner/revenue loss
- **Compliance**: If API traffic carries sensitive data, the RST handling must not cause data loss or failure of compliance logging. Ensure TLS is never downgraded on reset paths
- **Operational**: Document the gateway network configuration (worker count, timeouts, kernel tunables) in a runbook; test the effect of worker/scaling changes with load tests before rollout

## Senior-Level Answer

RST resets with healthy backends plus a recent worker-count change strongly suggest a kernel-level accept-queue overflow — especially if `tcp_abort_on_overflow=1`. I'd capture on the Kong node to confirm the RST comes from Kong, check NLB `TCP_Reset_Count`, inspect accept queue overflow counters and somaxconn, and review keepalive/idle timeouts. Fix: tune somaxconn/backlog, correct the worker count, maybe set `tcp_abort_on_overflow=0`, and align timeouts across the chain.

## Architect-Level Answer

This failure mode (kernel-generated RSTs after a config change) reveals a gap between the "application" and the "network" abstraction: resets can be generated below the app, invisible in application logs. Architecturally, I would instrument connection-level telemetry end to end: TCP exception metrics on the gateway hosts (resets, queue overflows, fd exhaustion), NLB reset metrics, and client-side RST telemetry, fed into a unified observability platform with SLO alerting. I would standardize gateway resource sizing in IaC (worker count, fd/rlimit, backlog, kernel tunables) so config changes are validated by load tests in staging before reaching prod. I would also decouple the gateway from ephemeral connection state where possible — using connection draining, graceful shutdown, and health-based deregistration so autoscaling never resets mid-request connections.

## Follow-Up Questions

1. How would you distinguish between an RST generated by Kong itself vs. an RST generated by the NLB, using only packet captures on the Kong node and NLB metrics?
2. If the resets correlate with a specific client that uses HTTP/2 over TLS, what additional layer would you inspect vs. HTTP/1.1 clients?
3. Under what circumstances would increasing the number of worker processes reduce connection resets rather than cause them, and how would you determine the right worker count?
4. How would you load-test the gateway to reproduce and measure a 2% reset rate before it goes to production?
5. If the RST is caused by an inline WAF matching a signature, how would you identify the matched traffic while respecting privacy and compliance constraints?