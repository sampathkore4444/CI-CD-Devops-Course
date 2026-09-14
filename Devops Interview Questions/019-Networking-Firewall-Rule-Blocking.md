# 019. Firewall Rule Blocking Application Traffic

## Scenario

After a security audit, firewall rules were updated on the network. Now a critical application cannot reach an external payment API on port 443. The connection times out. The application runs on a Kubernetes cluster in AWS, inside a private VPC subnet. The application talks to the external payment API at `https://payments.example.com` (public IP 203.0.113.50). The firewall that was updated is a network-level firewall device between the VPC and the internet (could be a network ACL, security group, or an on-premises firewall for hybrid environments). Since the connection times out (not connection refused), packets are either being dropped silently by the firewall (no RST returned) or route is not yet propagated. The security audit was performed last night and the firewall rules were updated by the network security team at 2 AM. The application team is not sure what rules were added or changed. The payment API integration is critical for order processing and has stopped working.

## Interviewer Question

After a security audit, firewall rules were updated on the network. Now a critical application cannot reach an external payment API on port 443. The connection times out. How do you verify if it's a firewall issue and get the traffic flowing?

## What I Should Think About

- Connection timeout (not refused) points to a firewall dropping packets, not the destination refusing the connection
- The recent firewall change is a strong leading indicator: the change was made last night and the failure started after it
- Need to verify the network path and whether the failure is at the AWS security group, VPC network ACL (NACL), or the external firewall
- Use tools to isolate: traceroute/mtr to see where the path stops responding, curl with verbose output, and TCP connection tests (nc/telnet)
- Check if the issue affects all traffic or just specific ports/protocols (e.g., only HTTPS to the payment API)
- Since the application runs in a private subnet, it may go through a NAT gateway or a transit gateway / VPN to the internet
- It could be an outgoing security group rule that was tightened during the audit
- The network ACL (NACL) in AWS is stateless and needs both inbound and outbound rules
- Check if the failure is DNS-related as well: "cannot reach payment API" could mean the DNS name resolves to a wrong IP, or the DNS query itself is blocked

## Ideal Answer

This is a clear case where traffic is being dropped, not refused. The timeout symptom tells us the packets are disappearing somewhere in the network path for HTTPS (port 443). The sequence I would follow:

1. Reproduce the failure from the application's perspective: `curl -v https://payments.example.com` from a pod and from the node
2. Confirm the destination IP and port: `dig +short payments.example.com`, then test the IP directly
3. Run `mtr`/`traceroute` from the node to the destination IP. Look at where the path stops responding
4. Check the Kubernetes node's outbound connectivity: does the node reach the internet? Does the node reach the payment API?
5. If the pod works but the node doesn't, the issue is the CNI and/or a pod-level policy. If the node works but the pod doesn't, it's a pod-level issue (security group, network policy, or SourceIP/Egress)
6. Check routing: `ip route get 203.0.113.50` from the node to see what interface/route is used
7. Check the AWS security groups and NACL rules on the relevant subnets/eni, and the AWS flow logs
8. Check if the application uses a NAT gateway for egress: the NAT gateway's security group/NACL may have dropped rules
9. Verify firewall logs for drops of the source IP/port to the payment API IP:443

Once the firewall is confirmed as the blocker (e.g., an outbound NACL rule or SG rule doesn't allow 443 to this destination), request the proper rule addition and validate.

## Architecture

```
Pod/Container
    │
    ▼
CNI veth
    │
    ▼
Node eth0 (private subnet)
    │  -- outbound traffic
    ▼
┌──────────────┐     ┌──────────────────┐     ┌──────────────┐
│ Security     │     │  VPC Network     │     │  NAT Gateway │
│ Group (SG)   │     │  ACL (NACL)      │     │ or Transit GW│
│ (stateful)   │     │  (stateless -    │     │ / VPN        │
│              │     │   needs in+out)  │     │              │
└──────────────┘     └──────────────────┘     └──────┬───────┘
                                                      │
              (possible intermediate firewall)
                                                      ▼
                                      ┌────────────────────────┐
                                      │  External Firewall     │
                                      │  (on-prem/3rd-party)   │
                                      │  -- UDP 443 allow?     │
                                      └───────────┬────────────┘
                                                  ▼
                                      External Payment API
                                      203.0.113.50:443
```

## Investigation

1. Reproduce from the pod and node levels to establish the scope
2. Confirm DNS resolution of the payment API hostname
3. Test connectivity to the IP directly (bypass DNS)
4. Run traceroute/mtr to understand the network path
5. Check routing on the node
6. Review security group rules for the node/pods and the egress path
7. Review NACL rules for the subnets
8. Check VPC flow logs
9. If NAT gateway is used, verify its elastic IP and check outbound rules
10. Check if any AWS WAF, transit gateway route table, or enterprise proxy introduces restrictions

## Commands

```bash
# Test from a pod
kubectl exec -it <app-pod> -- curl -v --connect-timeout 5 https://payments.example.com

# Test from the node (SSH into node or run from a debug daemonset)
node$ curl -v --connect-timeout 5 https://payments.example.com

# Check the IP and test directly
dig +short payments.example.com
# Expected: 203.0.113.50
curl -v --connect-timeout 5 https://203.0.113.50/

# TCP handshake test
node$ nc -zv 203.0.113.50 443
node$ telnet 203.0.113.50 443
# If connection times out, something is dropping SYN.

# Trace the network path
node$ traceroute -n -T -p 443 203.0.113.50   # TCP-based traceroute for HTTP(S)
node$ mtr -r -c 10 -T -P 443 203.0.113.50

# Check routing on the node
ip route get 203.0.113.50
ip rule show
ip route show table all | head -20

# Check interfaces and IPs
ip addr show eth0
curl ifconfig.me   # what is my public IP

# Check security group rules (If using AWS CLI)
aws ec2 describe-security-groups --group-ids <sg-id> --region <region> --query 'SecurityGroups[0].IpPermissions'

# Check NACL rules for the subnets
aws ec2 describe-network-acls --region <region> --filters Name=association.subnet-id,Values=<subnet-id>

# Check VPC flow logs for DENY/REJECT
# Athena query, or use CloudWatch Logs Insights:
# filter @logStream like /<eni-id>/
# | fields @timestamp, srcAddr, dstAddr, action, protocol
# | filter action = "REJECT" or action = "DENY"
# | sort @timestamp desc
# | limit 20

# Check if a NAT gateway is in the path
aws ec2 describe-route-tables --region <region> --filters Name=route.destination-cidr-block,Values=0.0.0.0/0
# Look for a NAT Gateway in the route table for the private subnet

# Check if there's an enterprise proxy in the path
node$ env | grep -i proxy
node$ cat /etc/resolv.conf

# Test protocol connectivity with openssl to confirm TLS layer
node$ openssl s_client -connect 203.0.113.50:443 -servername payments.example.com -brief

# Check with tcpdump on the node to see if SYN is sent and nothing returns
node$ tcpdump -i eth0 -n host 203.0.113.50 and port 443
node$ tcpdump -i eth0 -n 'tcp[13] & 0x02 != 0'    # watch for SYN packets
```

## Root Cause

- **Outbound NACL or SG rule removed**: The security audit removed "allow-all outbound" rules and introduced a deny-all-by-default policy, but the payment API IP:443 was not added to the allow list
- **Stateless NACL mismatch**: NACLs are stateless; the audit added outbound Allow 443 but did not add the corresponding inbound ephemeral-port Allow rules for return traffic (high port 1024-65535), causing replies to be dropped
- **Firewall rule ordering**: The payment API traffic matches a broader deny rule that was added above the more specific allow rule
- **Proxy / egress gateway change**: The audit introduced a new egress firewall/proxy that is not configured to allow HTTPS to the payment API
- **NSG / host firewall on nodes**: The audit updated node-level host firewall rules (iptables/ufw) that drop outbound traffic from pods

## Immediate Mitigation

1. If a NACL rule is dropped, add the missing outbound Allow and the corresponding return Allow as a quick fix, then validate
2. If a security group rule is missing, add the missing outbound 443 allow to the source IP of the payment API (or the target IP)
3. If an external firewall is the blocker, request an exception with a clear justification (requires communication with the network security team)
4. As an interim workaround, if traffic only needs to go to the payment API, temporarily route it through a different egress path (e.g., a different NAT gateway, proxy, or route) that is not blocked
5. Verify after the fix: curl from the pod and node, plus a TLS handshake test

## Permanent Fix

1. Review and update firewall rules using infrastructure-as-code (terraform) so they are version-controlled and audit-friendly
2. Implement automated connectivity testing after any firewall change: a smoke test that checks critical external endpoints (payment API, DNS, internal APIs)
3. Establish a change-control/review process for network security changes so application teams are notified before rules change
4. Document the full egress path for critical applications and keep a map of required firewall rules
5. Implement monitoring: external endpoint reachability checks
6. Keep NACL and SG rules paired correctly (stateless rule symmetry) - use a tool that validates rule symmetry

## Monitoring

- Implement synthetic HTTPS health checks to the payment API (curl or a health-check service) with alerts on failure
- Monitor VPC flow logs for newly denied traffic (feed into SIEM with alerts on DENY/REJECT to critical endpoints)
- Add multi-hop path checks: monitor reachability from pod, node, and through NAT gateway
- Alert on any change to security groups / NACLs (AWS Config rules)
- Track rate of denied outbound connections per security group
- Set up CloudWatch alarms on the NAT gateway's PacketsOutToDestination metric

## Security

- Ensure firewall rules follow least-privilege and are reviewed by a security team with application impact assessment
- Log all deny events and connect them to an incident-response workflow
- Ensure any change to egress rules involves a change ticket covering the affected applications
- Validate that the payment API integration allows only the necessary source IPs and protocol (TLS 1.2+)
- Do not introduce permissive allow-all rules in the name of a quick fix; add narrowly-scoped exceptions

## Production Considerations

- **High Availability**: Ensure the egress gateway / NAT gateway is HA (multiple AZs) or a transit gateway is provided with redundant paths
- **Scalability**: The egress NAT/gateway bandwidth must scale with the application's outbound traffic. Use multiple NAT gateways per AZ
- **Reliability**: Implement failover: if one egress path fails, traffic should route through a backup
- **Cost**: NACL/SG misconfigurations cause outage and recovery time; automation reduces both. Enforce IaC for all network rules
- **Compliance**: Document firewall exception requests. Maintain change history. Provide evidence of least-privilege to auditors
- **Operational**: Create a runbook for "external connectivity / egress" outages with checklists. Test firewall change pipelines in a staging environment

## Senior-Level Answer

The symptom of a timeout (no RST) combined with the change occurring during the security audit's firewall update points to a silent drop by a packet filter. I'd reproduce connectivity from the pod and the node, then walk the path: DNS, routing (`ip route get`), SG, NACL, flow logs, NAT gateway. The fix is to re-allow outbound 443 (and the matching return path) through the correct device, validate, then implement automated reachability checks so rule changes are caught before users or API integrations break.

## Architect-Level Answer

This incident exposes a process gap: network security changes were made without understanding impact on connected systems. Architecturally, I would separate "network engineering" from "network policy" and enforce that all external egress flows through a centrally-managed egress gateway where allow/deny rules are defined in code with the application's approval attached. I would place a proxy (or egress security gateway) in the path so rules are centrally enforced and auditable. I would also add automated connectivity checks to critical external dependencies (payment API, DNS, cloud APIs) run after every firewall/route change. In the long run, a policy-as-code approach (Open Policy Agent-style) for network rules with application-level approval workflow prevents this class of incident from recurring.

## Follow-Up Questions

1. How would your diagnosis differ if the connection returned "Connection refused" instead of a timeout?
2. If the application needs to reach 10 different external APIs, how would you prioritize which egress rules to establish and validate?
3. How do you test connectivity from a Kubernetes pod when `kubectl exec` and `curl` tools are not available inside the container?
4. If traffic is going through a NAT gateway and reaching the payment API, but the return traffic fails due to a NACL rule, what symptom would you observe?
5. How would you design a CI/CD pipeline that automatically validates firewall rules before deploying them to production?