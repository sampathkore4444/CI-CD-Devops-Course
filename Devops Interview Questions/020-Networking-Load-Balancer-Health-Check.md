# 020. Load Balancer Health Check Failures

## Scenario

An AWS ALB is marking all target instances as unhealthy. The application is running and responding correctly when accessed directly on the instance. Health checks are configured to hit the `/health` endpoint. The ALB target group is configured with HTTP health checks on port 80, path `/health`, interval 30 seconds, healthy threshold 5, unhealthy threshold 3. The application uses Nginx in front of a Python/Flask backend. Directly accessing `http://instance-ip/health` from the instance itself returns 200 OK. The health check has been failing for about 30 minutes, and Nginx and application logs show no errors. The instance's security group allows traffic on port 80 only from the ALB's security group. The issue started after a security group change was made to tighten rules. The ELB security group (SG) reference is used, and the load balancer has a security group too. Traffic to the application from clients through the ALB is returning 502 Bad Gateway errors because the target group has no healthy targets.

## Interviewer Question

An AWS ALB is marking all target instances as unhealthy. The application is running and responding correctly when accessed directly on the instance. Health checks are configured on /health endpoint. How do you troubleshoot and fix?

## What I Should Think About

- The app responds 200 when accessed directly on the instance, but ALB sees it unhealthy: the difference is in the network path between ALB and the instance, not the app itself
- The security group change is the prime suspect: the SG now only allows traffic from the ALB SG, but possibly the ALB SG reference is wrong, or the ALB's own SG isn't defined correctly, or the health check comes from a different source than expected
- Health checks from an ALB originate from the ALB's private ENIs (interfaces in the VPC), not a fixed public range. Using an SG reference (sg-xxxx) is the correct way to allow traffic
- Common causes:
  - The ALB SG reference in the target instance's ingress rule points to the wrong SG
  - The ALB's own security group doesn't allow outbound traffic on port 80 to the target
  - The target instance SG allows port 80 only from the ALB SG, but health checks may arrive from a different source IP
  - The target group's health check port is different from the app's listening port
  - The target group's health check path returns a 404 or a non-2xx/3xx code but returns 200 on the instance (e.g., a host header mismatch, trailing slash, or redirect)
  - The app returns 200 for the instance's public IP but the ALB's health check uses a different Host header
  - The NACL on the subnet is blocking traffic (if changed as part of the "security group change")
  - The ALB security group lacks an ingress rule from the subnet CIDR / the client SG
- Check whether clients are also timing out when they access the app through ALB (would confirm the SG block)

## Ideal Answer

The app returns 200 locally but the ALB sees it unhealthy. The difference is the network path from the ALB to the instance. Given the recent SG change, this is almost certainly a security group/NACL issue.

I would:
1. Confirm the health check returns 200 from the instance itself: `curl -I http://localhost/health` and `curl -I http://<instance-private-ip>/health`
2. Replicate the ALB's health check exactly: curl the health check endpoint using the path, port, and an appropriate Host header the way the ALB would
3. Check the ALB target group health check settings: port, path, interval, thresholds, protocol
4. Check the security groups: the instance's SG should allow inbound 80 from the ALB SG. The ALB SG should allow outbound 80 to the target instances (or at least not block it)
5. Verify the SG reference actually resolves to the ALB's SG (correct ID, correct region)
6. Check the subnet NACL on the instance subnet: since NACLs are stateless, both inbound and outbound rules must exist
7. Verify with flow logs or by checking netstat/ss on the instance during a health check to see if connections are coming in
8. Check if there's more than one target in the group (only some fail?) - "all unhealthy" vs "some unhealthy" tells us if it's instance-specific or common
9. Check the ALB's own security group for its ingress/egress rules

The fix: correct the SG rules to allow the ALB health check to reach the target and the target to respond.

## Architecture

```
Clients
   │
   ▼
┌─────────────────┐
│    ALB SG       │  (sg-alb-xxxx)
│  ingress :80/443 from client SG
│  egress  :80 to instance SG
└─────────────────┘
   │
   │  Health check traffic
   ▼
┌──────────────────────────────┐
│  Instance SG (sg-instance)   │
│  ingress :80 FROM sg-alb     │  ← must be the ALB SG ID
│  egress  :80 (ephemeral)     │
└──────────────────────────────┘
   │
   ▼
┌──────────────────────────────┐
│  NACL (subnet, stateless)    │
│  IN :100 - 80 from ALB subnet CIDR
│  OUT:100 - ephemeral 1024-65535
└──────────────────────────────┘
   │
   ▼
Instance (Nginx :80 → Flask :5000)
   /health → 200 OK (when requested directly)
```

## Investigation

1. Reproduce the health check from the instance itself
2. Replicate the health check with a Host header and exact path as the ALB would
3. Check the ALB target group health check configuration parameters
4. Examine instance SG rules and verify the ALB SG ID is referenced correctly
5. Examine ALB SG rules: outbound to the target, correct port
6. Check NACL on the target subnet (stateless rules symmetry)
7. Check for multiple targets: all vs. some unhealthy
8. Verify the ALB ENI and target IPs are in the same subnet/VPC
9. Check flow logs or use a tool like CloudWatch/Athena to see if health check requests reach the instance
10. If flow logs show the requests being rejected/dropped, correlate with SG/NACL rules

## Commands

```bash
# On the instance: test the health check endpoint locally
curl -I http://localhost/health
curl -s -o /dev/null -w "%{http_code}\n" http://localhost/health

# Test via the instance's private IP (from another instance in the same VPC)
curl -s -o /dev/null -w "%{http_code}\n" -H "Host: <domain>" http://<instance-private-ip>/health

# List target groups and health status
aws elbv2 describe-target-groups --region <region>
aws elbv2 describe-target-health --target-group-arn <target-group-arn> --region <region>
# Look for: State = unhealthy, Reason = Timeout/Connection refused/Code mismatch

# Show the exact health check configuration
aws elbv2 describe-target-groups \
  --target-group-arns <target-group-arn> \
  --query 'TargetGroups[0].{Port:Port,Protocol:Protocol,HealthCheckPort:HealthCheckPort,HealthCheckPath:HealthCheckPath,HealthCheckIntervalSeconds:HealthCheckIntervalSeconds,HealthyThresholdCount:HealthyThresholdCount,UnhealthyThresholdCount:UnhealthyThresholdCount}' \
  --region <region>

# Show the SG IDs attached to the ALB
aws elbv2 describe-load-balancers --load-balancer-arns <lb-arn> --query 'LoadBalancers[0].SecurityGroups' --region <region>

# Show the SG rules on the instance
aws ec2 describe-security-groups \
  --group-ids <instance-sg-id> \
  --query 'SecurityGroups[0].{Ingress:IpPermissions,Egress:IpPermissionsEgress}' \
  --region <region>

# Show the SG rules on the ALB
aws ec2 describe-security-groups \
  --group-ids <alb-sg-id> \
  --query 'SecurityGroups[0].{Ingress:IpPermissions,Egress:IpPermissionsEgress}' \
  --region <region>

# Check if there are route/NACL issues
aws ec2 describe-network-acls --region <region> --filters Name=association.subnet-id,Values=<subnet-id>
aws ec2 describe-route-tables --region <region> --filters Name=route.subnet-id,Values=<subnet-id>

# On the instance while the ALB is running a health check, check the connections arriving
# Watch for connections from the ALB private IPs on port 80
watch -n 5 'ss -ant | grep :80'

# Use tcpdump on the instance to see if health checks arrive and how they're handled
sudo tcpdump -i eth0 -n port 80

# Check Nginx access logs to see if health check requests arrive
tail -f /var/log/nginx/access.log | grep /health

# Check the app responds with proper status codes for the health path
curl -v http://localhost/health
# Confirm it returns exactly 200/2xx, not a redirect or 401

# Test from within the VPC from another instance that mimics the ALB (if possible)
# Source should be the same subnet CIDR as the ALB ENIs
curl -s -o /dev/null -w "%{http_code}\n" http://<target-private-ip>/health

# Check VPC flow logs (if enabled) to see if ALB->target connections are accepted or rejected
# query via Athena or log stream with the ENIs involved
aws ec2 describe-vpc-flow-logs --region <region>   # check if flow logs are enabled
```

## Root Cause

- **Instance SG does not reference the ALB SG**: The SG was changed to "allow port 80 from ALB SG" but the SG ID in the reference is wrong (or points to an SG that was deleted / different ENI)
- **ALB SG lacks egress to the target**: The ALB's SG allows clients in but its outbound rules block traffic to the target instances on port 80
- **NACL stateless mismatch**: The subnet NACL blocks inbound 80 from the ALB's subnet, or the NACL's return traffic rule is missing the ephemeral port range
- **Health check port/path mismatch**: ALB checks port 80 but Nginx listens on 8080, or the path doesn't actually exist on Nginx (e.g., expects a trailing slash or custom port that only works with a Host header)
- **Web application firewall / rate-limiting**: A rate limiter or WAF rule on Nginx drops health check requests while allowing other traffic
- **Multiple ENIs**: The instance responds on a different ENI/private IP than the ALB is health-checking

## Immediate Mitigation

1. Correct the SG rules: add an inbound rule on the target SG for port 80 from the ALB SG (correct ID), and outbound on the ALB SG to port 80 of target
2. If the NACL is blocking, add the missing stateless rule pair
3. Temporarily set the ALB health check to a simpler target (e.g., port with a bare `/` response) if the issue is the health check path or Host header
4. If the issue lasts, mark the target health manually (temporarily) or deregister/register to force a re-evaluation: `aws elbv2 deregister-targets --target-group-arn ... --targets Id=<instance-id>` then re-register
5. After the SG/NACL fix, wait 5-6 min (below unhealthy threshold is 3 × 30s = 90s; healthy after 5 × 30s = 150s) then confirm targets become Healthy

## Permanent Fix

1. Implement SG changes through IaC so the correct SG references are applied consistently
2. Add automated validation: after SG/NACL changes, run a connectivity smoke test to target instances from a health-check-simulating source
3. Enforce a rule that all ALB health checks point to a dedicated, stable `/healthz` endpoint that is not rate-limited or WAF-protected
4. Standardize SG naming and use tags (Name) to make SG references self-identifying
5. Set up AWS Config rules to detect SGs with rules referencing deleted or empty SGs
6. Document the expected SG rules between ALB and targets and enforce with drift detection

## Monitoring

- Monitor target group health: alert when any target is unhealthy or when healthy host count drops
- Track ALB 502/504 error rates: a drop to 0 healthy targets means 100% 502s
- Monitor Nginx access logs for /health status code distribution
- Set up CloudWatch alarms on TargetGroup "HealthyHostCount" below expected
- Add a synthetic check that curls the health endpoint from a separate source (CloudWatch Synthetics) so health is validated from outside the instance
- Track SG/NACL changes via CloudTrail and trigger alerts on network security changes

## Security

- Use the ALB SG ID (not IP ranges) in the target SG ingress rule to limit access to the load balancer only
- Don't open port 80 to 0.0.0.0/0 on target instances; keep SG references narrow and least-privilege
- Ensure health check path does not expose sensitive info or allow DoS via the health check itself
- Ensure the health endpoint doesn't perform expensive DB or cache operations (it should be lightweight)
- Keep NACL rules symmetric (stateless) and reviewed by a security team
- Do not allow the ALB's SG to be internet-routable with unrestricted egress; restrict egress rules

## Production Considerations

- **High Availability**: Run targets in multiple AZs so an AZ failure doesn't drop all targets. ALBs require at least 2 AZs
- **Scalability**: Set healthy/unhealthy thresholds appropriately to avoid flapping during auto-scaling or application restarts. Don't make them too aggressive
- **Reliability**: Health checks gate traffic; design them to reflect true readiness (readiness vs. liveness). The app must return 200 only when it's truly able to serve
- **Cost**: Health check misconfiguration causes 502s and traffic loss; invest in health check understanding to avoid flapping 
- **Compliance**: Health check logs and target health data may be relevant to audit; retain CloudTrail/flow logs for changes
- **Operational**: Use a standard health-check port and path across services. Document the health check behavior in the service runbook and developer onboarding

## Senior-Level Answer

An ALB marking targets unhealthy while the app returns 200 locally means the difference is in the path from ALB to the instance — almost always the SG reference, the ALB's own SG egress, or a stateless NACL mismatch. I'd reproduce the health check exactly, review the SG rules (both sides), review NACL symmetry, and check flow logs to see where the check is dropped. The fix is to correct the SG/NACL rule with the correct ALB SG ID, then wait for the health check interval to confirm Healthy.

## Architect-Level Answer

The recurring "ALB sees unhealthy, app is fine" scenario comes from a gap between network config and health-check semantics. Architecturally, I would make health checks part of the application contract: every service exposes a stable `/healthz` that returns 200 with empty body, is never rate-limited or behind WAF, and reflects readiness. I would standardize SG references via IaC so SGs are never edited manually, enforce drift detection (AWS Config) so a manual SG edit creates an alert, and add automated connectivity validation that runs whenever a network security change is merged. I would also move to an ALB per service with managed instance groups so health checks are consistent across the platform — the same contract applied everywhere.

## Follow-Up Questions

1. Some targets are healthy and some aren't. How would your diagnosis change compared to when ALL are unhealthy?
2. How do you distinguish between a "connection refused" health check failure and a "timeout" health check failure, and what does each imply?
3. If the health check succeeds from the instance but fails from the ALB due to a Host header mismatch, how would you detect that by only looking at the ALB and flow logs?
4. How would you design health checks for a service with a heavy startup time (>5 minutes) so flapping is avoided during rolling deployments?
5. If you mark the target manually as healthy to bypass the health check, what risks does that pose and how would you handle them?