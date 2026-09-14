# 65. VPC Peering Connectivity Issues

## Scenario

You're migrating a legacy monolith into a new VPC (`vpc-prod-app`) so it can talk to a shared database + secrets VPC (`vpc-shared-data`) in the same region. The two VPCs are peered (`vpc-0abc... ↔ vpc-0def...`). Route tables in both VPCs have entries for the peer (10.5.0.0/16 → pcx-xxxx, 10.6.0.0/16 → pcx-xxxx). The security groups on the DB instance and the app instances appear to allow the traffic. Yet the application in the new VPC cannot connect to the database (connection timeout) or reach the Secrets Manager endpoint in the shared VPC. Nothing talks to anything across the peer, but within each VPC everything works. The network team insists "the peering looks fine." Your job: identify the actual failure and fix it.

## Interviewer Question

"Two VPCs are peered, route tables are added, and security groups look correct — yet instances in one VPC can't reach instances in the other. What are the remaining possible causes, and walk me through your troubleshooting approach for VPC peering."

## What I Should Think About

- VPC peering is not transitive and doesn't propagate routes — each VPC must have explicit routes
- Peering allows traffic only if every layer allows: route table, SGs, NACLs, DNS naming, instance operating system
- Peering does NOT support/handle: overlapping CIDRs, transitive routing via IGW/NAT, IPv6 without config, Edge-to-Edge like VPN/Transit/IGW VPC-to-internet
- CIDR overlap is silent killer — if both VPCs use 10.0.0.0/16, routes conflict and nothing works
- DNS hostnames: by default, EC2 doesn't resolve private DNS of the peer without `enableDnsHostnames`, and Route 53 private zones need association/disclosure
- Check the peer status: `peering-connection-status` must be `active`
- Check the route `Propagate` flags — a VPC's main route against peer vs peering `pcx-`
- NACLs default-allow on both, but a custom NACL can silently block
- Instance-level host firewall (iptables / ufw / Windows firewall) often blocks cross-VPC on a specific port
- The endpoint in the shared VPC: if an app reaches a VPC Endpoint (SecretsManager) via its DNS, peering can't cross the VPC endpoint's private DNS unless region/interface conditions are met (interface endpoint DNS names are scoped per-VPC)
- Route propagation with VGW/TGW may hijack routes
- Check `aws ec2 describe-route-tables` for both VPCs — is there actually a route for the OTHER VPC's CIDR and does it point at `pcx-*`
- Local route for own CIDR: overlapping CIDRs get shadowed by the local route

## Ideal Answer

**Step 1 — Verify the peering is active**

```bash
aws ec2 describe-vpc-peering-connections \
  --vpc-peering-connection-ids pcx-0abc123 \
  --query 'VpcPeeringConnections[0].Status'
# Status must be "active". If "pending-acceptance" or "expired" → that's the problem.
```

**Step 2 — Verify routes both ways**

```bash
# App VPC routes
aws ec2 describe-route-tables \
  --filters Name=vpc-id,Values=vpc-0abc \
  --query 'RouteTables[].Routes[]'

# Shared VPC routes
aws ec2 describe-route-tables \
  --filters Name=vpc-id,Values=vpc-0def \
  --query 'RouteTables[].Routes[]'

# Look for: Destination = 10.6.0.0/16, Target = pcx-0abc123
```

Check for a **local route shadowing** — for overlapping CIDRs, the local route (VPC's own CIDR) takes precedence, making the peer supposedly unreachable.

**Step 3 — Check security groups and NACLs on both ends**

```bash
# SG on the DB instance
aws ec2 describe-instances --instance-ids i-db \
  --query 'Reservations[].Instances[].SecurityGroups[]'

aws ec2 describe-security-groups --group-ids sg-app sg-db

# NACL on both subnets
aws ec2 describe-network-acls --filters Name=vpc-id,Values=vpc-0abc --query 'NetworkAcls[].Entries[]'
```

**Step 4 — Connectivity tests that don't need app installed**

From an instance in VPC A:
```bash
# ICMP (may be dropped by SG — not definitive)
ping 10.6.1.100

# TCP connectivity: use nc/tcpdump
nc -zv 10.6.1.100 5432
tcpdump -n -i eth0 tcp port 5432

# DNS resolution of private hostnames
nslookup db.shared-data.internal  # requires DNS hostname resolution config
```

Eng: SSH into the DB host; check `ss -tlnp | grep 5432` and that Postgres is bound to 0.0.0.0 or the private IP.

**Step 5 — Repair**

```bash
# Accept a pending peering
aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id pcx-xxx

# Add missing route (app → shared)
aws ec2 create-route \
  --route-table-id rtb-app \
  --destination-cidr-block 10.6.0.0/16 \
  --vpc-peering-connection-id pcx-xxx

# Fix SG — allow from app SG (best practice: reference SG not CIDR)
```

## Architecture

```
    VPC Peering — both directions must be permissive

    VPC A (vpc-app) 10.5.0.0/16            VPC B (vpc-shared) 10.6.0.0/16
    ┌──────────────────────────────┐      ┌──────────────────────────────┐
    │ Subnet app web 10.5.1.0/24   │      │ Subnet data 10.6.1.0/24       │
    │  ┌───────────┐               │      │  ┌───────────┐               │
    │  │ EC2 app   │──▶ pcx-xxxx ──┼──────┼─▶│ EC2 db    │               │
    │  └───────────┘               │      │  └───────────┘               │
    │ Route: 10.6.0.0/16 → pcx     │      │ Route: 10.5.0.0/16 → pcx     │
    │ SG: allow 5432 from sg-db    │      │ SG: allow 5432 from sg-app    │
    │ NACL: allow tcp 5432         │      │ NACL: allow tcp 5432          │
    └──────────────────────────────┘      └──────────────────────────────┘

    Failure points (each must be true for traffic to pass):
    1. pcx-xxxx status = active
    2. Route A → 10.6.0.0/16 via pcx
    3. Route B → 10.5.0.0/16 via pcx
    4. SG on A allows outbound (ephemeral ports)
    5. SG on B allows inbound 5432 from A
    6. NACL on both subnets allows both directions (stateless!)
    7. No CIDR overlap
    8. DNS/private hostname resolution works (if using DNS names)
    9. OS-level firewall (ptables/ufw) allows the port
```

## Investigation

1. **Peering status**: is it `active`? (pending-acceptance/expired/provisioning)
2. **Route tables**: both VPCs have a route for the OTHER VPC's CIDR, targeting `pcx-*`?
3. **CIDR overlap**: do the two VPC CIDRs overlap? (then routes conflict/shadow)
4. **SGs on both instances**: inbound on target from source's CIDR/SG
5. **NACL on both subnets**: NACLs are stateless — need explicit allow for both directions
6. **DNS naming**: cross-VPC instance hostnames (`ip-10-6-1-100.ec2.internal`) resolve only if `enableDnsHostnames` on both and using a TGW/Route53 Resolver; otherwise use IPs
7. **Endpoint cases**: a VPC endpoint (Interface endpoint) in B can't be reached by A through plain peering DNS vs. the endpoint's zone; solution = Route 53 private zone association
8. **OS firewall**: host-level `ufw`/iptables on the target instance
9. **App config**: connection string may point at the wrong hostname/IP
10. **Propagated routes**: a VPN/VGW/Transit Gateway route could be hijacking the peer route

## Commands

```bash
# Peering status
aws ec2 describe-vpc-peering-connections \
  --vpc-peering-connection-ids pcx-0abc123 \
  --query 'VpcPeeringConnections[0].Status'

# Both route tables side by side
aws ec2 describe-route-tables \
  --filters Name=vpc-id,Values=vpc-0abc \
  --query 'RouteTables[].Routes[].{Dest:DestinationCidrBlock,Target:GatewayId,Peer:VpcPeeringConnectionId}'
aws ec2 describe-route-tables \
  --filters Name=vpc-id,Values=vpc-0def \
  --query 'RouteTables[].Routes[].{Dest:DestinationCidrBlock,Target:GatewayId,Peer:VpcPeeringConnectionId}'

# SGs
aws ec2 describe-security-groups --group-ids sg-data --output table

# NACLs
aws ec2 describe-network-acls --filters Name=vpc-id,Values=vpc-0def \
  --query 'NetworkAcls[].Entries[].{Num:RuleNumber,Proto:Protocol,Action:RuleAction,Egress:Egress,Cidr:CidrBlock,Range:PortRange}'

# DNS/hostname settings
aws ec2 describe-vpc-attribute --vpc-id vpc-0abc --attribute enableDnsHostnames

# From the app instance — verify listener exists on DB
aws ssm send-command --instance-ids i-db \
  --document-name "AWS-RunShellScript" \
  --parameters 'commands=["ss -tlnp | grep 5432","ufw status","iptables -L -n | grep 5432"]'

# Modify SG (example: allow DB port from app CIDR)
aws ec2 authorize-security-group-ingress \
  --group-id sg-data --protocol tcp --port 5432 --cidr 10.5.0.0/16
```

## Root Cause

| Root Cause | Evidence | Fix |
|---|---|---|
| Peering not accepted | status: pending-acceptance/expired | Accept peering |
| Missing route one direction | Route table lacks other VPC CIDR | Add route via pcx |
| Route shadows/missing propagation | Local route with same CIDR wins | Change CIDR or use TGW with non-overlapping |
| CIDR overlap | Both VPCs use 10.0.0.0/16 | Re-CIDR one VPC |
| SG blocks | No inbound rule / wrong CIDR | Add source SG/CIDR rule |
| NACL blocks (stateless) | NACL entry missing outbound side | Add allow both directions on both U+down |
| VPC endpoint private DNS | Secret Manager endpoint reachable only in its own VPC zone | Associate Route53 private zone / use interface endpoint with DNS name |
| OS firewall | `ufw`/iptables drop the port | Allow port on host firewall |
| DNS resolution across VPCs | Hostnames resolve for local only | Enable DNS hostnames + Route53 Resolver / use IPs |
| TransitGateway/VPN route won | Propagated route has lower preference | Prioritize Propagate (static > propagated) / adjust TGW |

## Immediate Mitigation

1. **If peering is pending-acceptance**, accept it right now:
   ```bash
   aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id pcx-xxx
   ```
2. **If a route is missing**, add the missing route so traffic flows, then adjust SGs after connectivity is proven.
3. **If it's a VPC Endpoint DNS issue**, switch the app briefly to use the private IP of the endpoint ENI (fastest) while you fix the zone.
4. **If rescue is impossible quickly**, place a temporary EC2 proxy/NAT instance (iptables DNAT) in the target VPC to restore data flow within minutes.
5. **Validate the moment service is restored**: run the app's dependency `curl`/DB `nc` probe.

## Permanent Fix

1. **Standardize CIDR blocks** — never overlap; plan a central IPAM (e.g., allocate from AWS IPAM, reserved ranges)
2. **Central route management via IaC** — Terraform module creating the peering + both route entries + SG references atomically (no manual console work)
3. **Use SGs referenced by SG ID, not CIDR**, in cross-VPC rules — sane when network IDs change
4. **Prefer AWS PrivateLink / VPC Endpoints (interface) over peering for data-plane access to shared services** (Secrets Manager, database): no route-table/SG/NACL churn; service discovery stays in-zone
5. **Consider Transit Gateway** at scale (many VPCs) instead of a mesh of peerings; TGW handles用时 route propagation automatically
6. **Route tables per subnet with explicit peer routes; avoid main-route-table traps**
7. **Add peering health checks** in CloudWatch (VPC Reachability Analyzer schedules)

## Monitoring

```yaml
# Reachability Alert — use VPC Reachability Analyzer periodically
- name: VPCReachabilityAnalyzerJob
  default: daily
  action: Check source=app:5432 → dest=db:5432 and alert on FAIL

# Packet/session alarms via VPC Flow Logs
- alert: CrossVPCDroppedTraffic
  expr: sum by (srcaddr) (flowlogs{action="REJECT", src_vpc="vpc-0abc"}) > 0
  for: 5m
  severity: warning
```

Enable **VPC Flow Logs** on both VPCs (all traffic) to observe REJECT vs ACCEPT — this is the decisive artifact for cross-VPC diagnosis.

## Security

- Peering is a flat, bidirectional trust model — evaluate risk of widening the blast radius initially
- Use **AWS PrivateLink over peering for sensitive services**; peering exposes entire VPC CIDR
- Map ACL with `PrincipalServices, Allowed Ip addresses` around peer CIDR — no 0.0.0.0/0 effectively
- **Guard both NACLs stateless in/out**; log REJECTs for security visibility
- Apply **Route53 Resolver DNS Firewall** so cross-VPC DNS doesn't leak/query public
- Monitor for lateral movement across peering (GuardDuty on both VPCs)

## Production Considerations

- **Cost**: Peering is free (data transfer between VPCs is not — check VPC to VPC inter-data transfer); PrivateLink endpoint costs ~$0.01/hr + data
- **Limits**: 125 peering connections per VPC; no transitive routing; no overlapping CIDR
- **HA**: peering has no failover — if you outgrow it, Transit Gateway with VPN/attachment redundancy
- **Reliability**: auto-mitigate route churn with IaC; test connectivity script in CI/CD pipeline pre-deploy
- **Operational**: reachability-analyzer scheduled checks + on-call runbook
- **Compliance**: flow logs prove access; map app → db dependency for audits

## Senior-Level Answer

"For a cross-VPC connection failing, I check the four layers in order: peering status, routes, security groups/NACLs, then application/OS. The peering status must be `active`. Then I verify both VPCs have a route to the *other* VPC's CIDR targeted at `pcx-*`, and critically that there's no CIDR overlap shadowing that with the local route. If routes are fine, I compare SGs on both ends and remember NACLs are stateless — the outbound allow is mandatory, not just the inbound. I use VPC Flow Logs to see REJECT vs ACCEPT, which pinpoints whether it's SG, NACL, or routing. The sneaky one is DNS: a private hostname or a VPC-endpoint private DNS name will not resolve across a plain peering. Fix is Route 53 private zone association or use the raw IP. In urgent cases I switch to PrivateLink for the endpoint, which eliminates all this churn."

## Architect-Level Answer

"VPC peering is a connectivity primitive with real operational pitfalls (non-transitive routes, no automatic updates, CIDR shadowing). At architecture level, I treat intra-region shared data access as a service boundary: prefer AWS PrivateLink interface endpoints so the DB and Secrets Manager present stable DNS names inside every consuming VPC, with no route-table or NACL coordination. When peering is required, I encode it in infrastructure-as-code as an atomic unit — peering + both routes + SG references — and reject overlapping CIDR allocations via a central IPAM. For many-VPC estates I adopt Transit Gateway for route automation. I add VPC Flow Logs + Reachability Analyzer as the observability backbone so a future peering issue is answerable in minutes, not hours. The design goal: connectivity choices are declarative, auditable, and testable — not a maze of click-ops route tables."

## Follow-Up Questions

1. "Why can't you use a VPC peering connection to route traffic through a NAT gateway, internet gateway, or VPN in one of the VPCs?"
2. "If both VPCs use 10.0.0.0/16, describe exactly what happens when an instance in one pings an instance in the other."
3. "What's the difference between a VPC Endpoint (interface) and classic peering for access control? When is PrivateLink the better architectural choice?"
4. "How would you use VPC Flow Logs and AWS Reachability Analyzer to prove where a packet is being dropped in a peering path — give the exact workflow."
5. "You need to connect 15 VPCs together with minimal duplication of routes. Compare peering, transit gateway, and private endpoints in terms of cost, maintenance, and security blast radius."