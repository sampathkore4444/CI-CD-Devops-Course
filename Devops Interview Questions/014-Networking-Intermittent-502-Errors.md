# 014. Application Intermittent 502 Bad Gateway Errors

## Scenario

Your e-commerce platform is experiencing intermittent 502 Bad Gateway errors. The architecture consists of client requests hitting an AWS Application Load Balancer (ALB), which forwards traffic to Nginx reverse proxy servers, which in turn forward requests to the backend application servers running on port 8080. The errors affect approximately 5% of all requests and occur randomly with no clear pattern. The issue started after a deployment 3 days ago that increased the number of backend application pods from 10 to 25. Backend servers appear healthy in monitoring dashboards, CPU and memory are within normal ranges, and direct curl requests to backend servers return 200 OK. The error rate spikes slightly during peak hours but remains non-zero even during low traffic periods. Your team has been unable to identify the root cause despite checking application logs, which show no errors during the 502 events.

## Interviewer Question

Users are reporting intermittent 502 Bad Gateway errors from an application behind an Nginx reverse proxy and an AWS ALB. The errors happen randomly, about 5% of requests. Backend servers appear healthy. How do you diagnose whether it's the ALB, Nginx, the application, or the network?

## What I Should Think About

- A 502 Bad Gateway means the upstream server (Nginx) received an invalid response from the backend, or the backend closed the connection unexpectedly
- The intermittent nature suggests a race condition, connection pool exhaustion, or timeout mismatch rather than a complete failure
- The recent scaling event from 10 to 25 pods could have introduced new issues such as connection pool limits being hit, different pod configurations, or networking changes
- Need to systematically isolate each layer: ALB -> Nginx -> Backend Application -> Network
- Check for connection timeouts, keep-alive settings, upstream configuration, and request timeout mismatches
- Examine whether the issue correlates with specific backend pods, specific time windows, or specific request types
- Consider whether health checks are properly configured and whether new pods are being included in the load balancer before they are fully ready
- Check Nginx error logs for upstream connection refused, timeout, or reset errors
- Investigate whether the backend application has a connection limit per worker or per process

## Ideal Answer

The first step is to systematically isolate each component in the request chain. Start by examining Nginx error logs, as 502 errors originate from Nginx itself. Look for specific upstream error patterns such as "upstream prematurely closed connection," "connection refused," or "upstream timed out." These patterns will immediately narrow down the root cause.

Next, check Nginx upstream configuration for timeout settings. A common issue is that Nginx's proxy_read_timeout is shorter than the backend's processing time, causing Nginx to close the connection before the backend responds. Conversely, if the backend's connection timeout is too short, the backend may close idle connections that Nginx expects to be alive.

Then investigate connection pool settings. After scaling from 10 to 25 pods, if Nginx is configured with a fixed upstream connection pool (keepalive connections), it may be exhausting connections. Check the `keepalive` directive in the upstream block and the `keepalive_timeout` settings.

Also verify that the ALB health check configuration matches what the application actually exposes. If the ALB is marking pods as healthy before they are ready to serve traffic, requests routed to unready pods will fail. Check the ALB target group health check path, port, and healthy/unhealthy threshold counts.

Finally, check for socket exhaustion on the Nginx servers. If all backend connections are in use, new connections will be rejected, leading to 502 errors.

## Architecture

```
Clients
  │
  ▼
┌─────────────────────┐
│   AWS ALB           │
│  (Target Group)     │
│  Health: /health    │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│   Nginx Reverse     │
│   Proxy Servers     │
│   (Port 80/443)     │
│                     │
│  proxy_pass to:     │
│  upstream block     │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  Backend App Pods   │
│  (Port 8080)        │
│  25 replicas        │
│                     │
│  /health endpoint   │
└─────────────────────┘
```

## Investigation

1. Check Nginx error logs for upstream error patterns and categorize the type of 502
2. Examine Nginx access logs to see if 502 errors correlate with specific upstream servers, request paths, or time windows
3. Check Nginx upstream configuration for timeout and keepalive settings
4. Review ALB access logs to verify request routing and response codes
5. Verify ALB health check configuration and target group health status
6. Check backend application logs during the 502 time windows
7. Examine network metrics between Nginx and backend pods (connection resets, retransmits)
8. Check Nginx connection state using `ss` or `netstat` during error periods
9. Compare backend pod configurations between old (10 pod) and new (25 pod) deployments
10. Run continuous curl tests from Nginx to backend to reproduce and capture errors

## Commands

```bash
# Check Nginx error logs for upstream issues
tail -f /var/log/nginx/error.log | grep -i "upstream\|502\|connection"

# Look for specific 502 error patterns
grep "upstream prematurely closed connection" /var/log/nginx/error.log | tail -20
grep "connection refused" /var/log/nginx/error.log | tail -20
grep "upstream timed out" /var/log/nginx/error.log | tail -20

# Check Nginx access logs for 502 responses
awk '$9 == 502' /var/log/nginx/access.log | tail -20

# Check which upstream servers are getting 502s
awk '$9 == 502 {print $NF}' /var/log/nginx/access.log | sort | uniq -c | sort -rn

# Verify Nginx upstream configuration
nginx -T 2>/dev/null | grep -A 20 "upstream"

# Check current connection states on Nginx
ss -s
ss -ant | head -20
ss -ant | grep ESTABLISHED | wc -l
ss -ant | grep TIME-WAIT | wc -l
ss -ant | grep CLOSE-WAIT | wc -l

# Check Nginx worker connection usage
cat /proc/$(pgrep -o nginx)/limits | grep "Max open files"

# Monitor connections to backend in real-time
watch -n 1 'ss -ant | grep :8080 | awk "{print \$1}" | sort | uniq -c'

# Test backend directly from Nginx server
curl -v http://backend-pod-ip:8080/health
for i in $(seq 1 100); do curl -s -o /dev/null -w "%{http_code}\n" http://backend-pod-ip:8080/api/endpoint; done | sort | uniq -c

# Check ALB target health
aws elbv2 describe-target-health --target-group-arn arn:aws:elasticloadbalancing:...

# Check ALB access logs
aws s3 ls s3://alb-logs-bucket/AWSLogs/account-id/elasticloadbalancing/ --recursive | tail -5
# Download and analyze ALB logs
zcat ALB-logs.gz | awk '{print $14, $15, $16}' | sort | uniq -c | sort -rn

# Check backend pod readiness and restart counts
kubectl get pods -o wide | grep backend
kubectl describe pod <pod-name> | grep -A 5 "Readiness\|Restart\|Last State"

# Check network connectivity between Nginx and backend pods
kubectl exec -it nginx-pod -- curl -v http://backend-service:8080/health
kubectl exec -it nginx-pod -- nc -zv backend-pod-ip 8080

# Check for connection resets at the network level
netstat -s | grep -i "reset\|timeout\|retransmit"

# Monitor real-time connections during error reproduction
watch -n 0.5 'netstat -an | grep :8080 | awk "{print \$6}" | sort | uniq -c'
```

## Root Cause

- **Connection pool exhaustion**: Nginx upstream keepalive connections are all in use, new connections are refused. The scaling from 10 to 25 pods increased load on each Nginx server but the keepalive pool size was not adjusted.
- **Timeout mismatch**: Nginx `proxy_read_timeout` (default 60s) is shorter than the backend's maximum processing time for certain requests, causing Nginx to close the connection prematurely.
- **Backend connection limit**: Each backend pod has a maximum number of concurrent connections (e.g., Tomcat maxThreads, Node.js maxConnections). With more pods, each Nginx server maintains more upstream connections, and pods with lower limits start refusing connections.
- **ALB idle timeout**: The ALB idle timeout (default 60s) may be shorter than the Nginx proxy timeout, causing the ALB to close the connection before Nginx receives the full response.
- **Backend socket exhaustion**: The backend pods have too many connections in TIME_WAIT or CLOSE_WAIT state, preventing new connections from being established.

## Immediate Mitigation

1. Restart Nginx to clear any stuck connections: `systemctl restart nginx` (do this on one server at a time in a rolling fashion)
2. Increase Nginx upstream keepalive connections temporarily: Add `keepalive 256;` in the upstream block
3. Reduce Nginx proxy timeouts to fail fast: Set `proxy_read_timeout 30s;` and `proxy_connect_timeout 5s;`
4. If the issue is backend connection limits, increase the limit on the application side or reduce the keepalive connection pool
5. Scale up Nginx servers temporarily to distribute the connection load
6. If specific backend pods are causing issues, remove them from the upstream pool or mark them unhealthy in the ALB target group

## Permanent Fix

1. Tune Nginx upstream keepalive settings to match the number of backend connections needed: set `keepalive` to a value that allows sufficient connections without exhaustion
2. Align timeout values across the entire chain: ALB idle timeout > Nginx proxy_read_timeout > backend processing time
3. Configure proper connection pooling in Nginx using `keepalive` and `keepalive_timeout` directives
4. Set backend application connection limits appropriately based on expected load from each Nginx server
5. Implement proper upstream health checks in Nginx (nginx_upstream_check_module or commercial Nginx Plus)
6. Add connection-level monitoring to detect pool exhaustion before it causes errors
7. Review and adjust ALB health check thresholds to prevent routing to unready pods

## Monitoring

- Track 502 error rate per Nginx server and aggregate across all servers
- Monitor Nginx upstream connection states: active, idle, waiting
- Track connection pool utilization (connections in use vs. max configured)
- Monitor backend pod connection counts and response times
- Set up ALB target group health check status monitoring
- Alert on 502 error rate exceeding 1% for more than 2 minutes
- Monitor TCP connection states: TIME_WAIT, CLOSE_WAIT, ESTABLISHED counts
- Track ALB latency vs. Nginx latency to detect timeout mismatches

## Security

- Ensure ALB SSL termination is properly configured with strong cipher suites
- Verify that Nginx is not exposing internal upstream server details in error pages
- Check that rate limiting is configured on Nginx to prevent connection pool exhaustion from DDoS
- Ensure ALB security groups only allow traffic from expected sources
- Verify that backend servers are not directly accessible from the internet (only through Nginx/ALB)
- Check for any IP-based access restrictions on backend pods that might cause intermittent connection refusals

## Production Considerations

- **High Availability**: Deploy Nginx across multiple AZs to prevent single points of failure. Use upstream least-connections or ip-hash load balancing
- **Scalability**: When scaling backend pods, also consider scaling Nginx servers proportionally. Connection pool sizes must scale with backend pod count
- **Reliability**: Implement circuit breaker patterns in Nginx to fail fast when backends are unhealthy rather than queuing requests
- **Cost**: Over-provisioning Nginx servers is cheaper than losing revenue from 502 errors. Monitor connection metrics to right-size the fleet
- **Compliance**: Ensure ALB and Nginx access logs are retained per compliance requirements. Log all 502 events with full request context
- **Operational**: Create runbooks for 502 error scenarios. Automate Nginx configuration validation with `nginx -t` before deployment. Use configuration management to ensure Nginx settings are consistent across all servers

## Senior-Level Answer

I would start by examining Nginx error logs to classify the specific type of 502 error (connection refused vs. premature close vs. timeout). Then I would check the Nginx upstream configuration, focusing on keepalive connections and timeout values. Since the issue started after scaling from 10 to 25 pods, I suspect the connection pool is being exhausted or there is a timeout mismatch between Nginx and the backends. I would verify by monitoring Nginx connection states during peak error periods and correlating with specific backend pods. The fix typically involves tuning keepalive pool sizes, aligning timeout values across the ALB-Nginx-backend chain, and verifying backend connection limits are adequate.

## Architect-Level Answer

This is fundamentally a capacity planning and configuration drift problem exposed by a scaling event. The architecture lacks proper circuit breaking and connection pooling configuration that scales with backend instances. I would implement a service mesh or ingress controller (such as NGINX Ingress Controller or Istio) that provides dynamic upstream management with automatic health checking and connection pool tuning. Additionally, I would establish configuration baselines for all timeout and connection pool values across the infrastructure, enforce them through infrastructure as code, and implement canary deployment strategies that include connection-level health metrics before full rollout. The monitoring stack should include connection pool utilization alerts to prevent this class of issues from recurring.

## Follow-Up Questions

1. How would you handle this situation if you couldn't restart Nginx due to in-flight transactions that must not be dropped?
2. What is the difference between a 502, 503, and 504 error in the context of Nginx, and how would your troubleshooting approach differ for each?
3. How would you implement connection pooling in Nginx to handle 100 backend pods without exhausting file descriptors?
4. If the 502 errors only occurred on POST requests with large payloads, how would your diagnosis change?
5. How would you design a load testing strategy to validate that your Nginx and ALB configuration can handle a 3x traffic spike after the scaling change?
