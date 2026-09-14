# 021. VPN Connection Dropping Intermittently

## Scenario

A site-to-site VPN between an on-premises data center and AWS keeps dropping every few hours. During the drops, the application experiences brief outages. The VPN carries database replication traffic between the on-premises primary database and the AWS disaster recovery replica. The VPN is an IPsec site-to-site connection (AWS Virtual Private Gateway to an on-premises firewall). The drops last between 30 seconds and 2 minutes. After the drop, the tunnel re-establishes on its own (or a monitoring alert triggers a manual re-establishment). The database replication uses native PostgreSQL streaming replication over TCP port 5432. Replication gaps grow during each drop and catch up after the tunnel is re-established. The VPN logs show "Phase 2 expired" and re-key events. The on-premises firewall is a Cisco ASA. AWS is configured with two tunnels (failover). No hardware or provider incident is reported. This pattern has been going on for one week.

## Interviewer Question

A site-to-site VPN between on-premises and AWS keeps dropping every few hours. During drops, the application experiences brief outages. The VPN connects two data centers for database replication. How do you diagnose and stabilize the connection?

## What I Should Think About

- Site-to-site IPsec tunnels drop when the IKE (Phase 1) or IPsec SA (Phase 2) lifetime expires; the tunnel re-establishes after re-key. "Drops every few hours" points to a re-key/lifetime issue or a parameter mismatch
- Common causes: IKE/SA lifetime mismatch, dead peer detection (DPD) settings not aligned, NAT traversal issues, MTU/fragmentation on the path, one side's firewall simplifying the SA (e.g., ASA with a broad subnet selector), third-party device's idle timeout killing the tunnel (e.g., a middlebox), or a routing/flapping issue post-rekey
- Since the outage is brief (30s-2min), the tunnel recovery involves re-negotiation which takes time; if the application's TCP connections were dropped they need to re-establish
- In AWS: two tunnels per VPN (Tunnel 1, Tunnel 2). If both drop at once, it's a shared issue; if only one drops and the other takes over, we have failover working but still need to fix the root flapping
- For database replication, we care about: TCP sessions on port 5432, the replication slot, the LSN/replication gap, recovery time
- Key metrics to look at: tunnel state over time, re-key events, SA lifetime, DPD failures, uptime of tunnel phases, latency/RTT over the tunnel, packet loss on the tunnel path
- Need to check: AWS VPN tunnel details (from the AWS console/CLI), the on-prem firewall config (phase 1/2 lifetimes, DPD), the router's interface statistics, and monitoring/log captures on both ends

## Ideal Answer

I would establish what exactly "dropping" means — full Phase 1 (IKE) loss vs. just Phase 2 (IPsec SA) re-key, and whether both tunnels drop simultaneously. Then I would compare lifetimes, DPD settings, and timeout policies on both ends.

Steps:
1. Define the tunnel topology: both tunnels, their local/remote endpoints, and which carries the replication. Confirm whether only one drops or both
2. Collect IKE/IPsec logs from the on-prem firewall during a drop window: look for "SA re-key" vs. "Peer not responding" vs. "DPD timeout" events
3. Check AWS VPN tunnel status: `aws ec2 describe-vpn-connections` shows tunnel state; look at CloudWatch metrics `TunnelState` / `TunnelDataIn`/`TunnelDataOut`
4. Compare Phase 1 and Phase 2 SA lifetime values on both sides. If ASA says Phase 2 lifetime 28800s (8h) but AWS says 3600s (1h) or something different, re-key events will differ. If the ASA kills the SA at its lifetime while AWS still thinks it's valid, packets get dropped until re-negotiation
5. Check dead peer detection (DPD) / keepalive settings: AWS default DPD is different from a Cisco ASA default. If DPD reboot/drop detection is too aggressive, transient network jitter triggers tunnel restarts
6. Test the network path: `mtr` / `ping` from on-prem to AWS VPN endpoint, look at packet loss and latency spikes right before each drop (a jittery uplink or carrier issue could cause DPD timeout)
7. Check MTU/fragmentation: database streaming can carry large packet payloads; if MTU mismatch exists, some packets get fragmented and dropped, which can corrupt the TCP stream and cause it to reset during the re-key window
8. Check the application side: PostgreSQL streaming replication — after the tunnel drops, the WAL streaming connection dies; it reconnects automatically, but if `wal_keep_segments` or the replication slot is too small, the gap grows
9. Fix the mismatch (lifetime / DPD / MTU), and monitor to confirm the drop pattern disappears

## Architecture

```
                ┌──────────────────────────┐
                │         On-prem          │
                │   Cisco ASA / firewall   │
                │   (IKEv1/IKEv2)          │
                └───────────┬──────────────┘
                            │  Phase 1 IKE SA (UDP 500/4500)
                            │  Phase 2 IPsec SA (ESP 50)
                            │  Tunnel 1 + Tunnel 2
                            ▼
                ┌──────────────────────────┐
                │   AWS Virtual Private    │
                │   Gateway (VGW)          │
                │   Tunnel 1: IP1          │
                │   Tunnel 2: IP2          │
                └───────────┬──────────────┘
                            │
                            ▼
                ┌──────────────────────────┐
                │  AWS DR subnet           │
                │  PostgreSQL replica      │
                │  (streaming replication  │
                │   over TCP :5432)        │
                └──────────────────────────┘

   Primary DB (on-prem) ──── WAL stream ──── Replica (AWS)
   (replication gap when tunnel drops)
```

## Investigation

1. Determine drop scope: single tunnel or both tunnels, correlate with timestamps
2. Check AWS VPN tunnel status and uptime over the last week
3. Review on-prem firewall logs around the drop times
4. Check IKE/SA lifetimes on both ends
5. Check DPD settings and dead-peer detection logs
6. Test the network path to the AWS VPN endpoint for packet loss/jitter before and during drops
7. Check MTU across the path (ping -M do) and look for fragmentation
8. Monitor PostgreSQL replication gap, last WAL write time, and restart events
9. Check if there's a middlebox/NAT device between on-prem and the internet that is idle-timeout killing the tunnel
10. Test with both tunnels independently (temporarily disable one) to see if it's a shared or per-tunnel characteristic

## Commands

```bash
# Check AWS VPN connection details and tunnel states
aws ec2 describe-vpn-connections --region <region>
aws ec2 describe-vpn-connections \
  --vpn-connection-ids <vpn-id> \
  --query 'VpnConnections[0].VgwTelemetry' \
  --region <region>   # shows TunnelUPTimeSeconds, Status, LastStatusChange, OutsideIpAddress

# Monitor tunnel status over time (poll repeatedly)
watch -n 30 'aws ec2 describe-vpn-connections --vpn-connection-ids <vpn-id> --query "VpnConnections[0].VgwTelemetry[].{Address:OutsideIpAddress,Status:Status,LastStatusChange:LastStatusChange,UpTime:TunnelUPTimeSeconds}" --region <region>'

# Check CloudWatch metrics for the VPN tunnel
aws cloudwatch get-metric-statistics \
  --namespace AWS/VPN \
  --metric-name TunnelState \
  --dimensions Name=VpnId,Value=<vpn-id> \
  --start-time <date> --end-time <date> --period 300 \
  --statistics Average --region <region>

# On the on-prem firewall (Cisco ASA), check SA lifetimes
show crypto ikev1 sa detail
show crypto ipsec sa
show crypto isakmp sa
show run crypto map
show run crypto isakmp

# Check DPD settings
show run | include dcd  # or 'dead peer detection'
# AWS default DPD: IKEv1 10s / 3 retries

# Check re-key/lifetime on AWS side (from VPN connection config review)
# The configuration file AWS provides contains phase 1/2 parameters
# cat the VPN config that AWS generated:
#   diffie-hellman-group, encryption, hash,
#   IKE lifetime, IPsec lifetime, DPD

# Test the WAN path to the AWS VPN public endpoint (outside IP)
mtr -r -c 30 <aws-vpn-outside-ip>
ping -i 0.5 <aws-vpn-outside-ip>

# Test MTU on the path (jumbo vs normal)
ping -M do -s 1400 <aws-vpn-outside-ip>
ping -M do -s 1472 <aws-vpn-outside-ip>
# if 1472 doesn't work but 1400 does, MTU mismatch on the path

# Check for NAT traversal issues on port 4500
# (UDP 4500 must be open both directions through the firewall/middlebox)

# On the on-prem firewall, capture during a drop
# on ASA: capture vpn_cap type raw-data interface outside match ip any <aws-ip>

# Check PostgreSQL replication status after a drop
# On the server:
select * from pg_stat_replication;
select state, client_addr, sync_state, replay_lsn - sent_lsn as lag
from pg_stat_replication;

# Check WAL gaps / replication lag after a drop
select * from pg_stat_wal_receiver;
select now() - pg_last_xact_replay_timestamp() as replication_lag;

# Check PostgreSQL logs for WAL streaming connection drops
grep -i "could not receive data from WAL stream\|connection timeout" postgresql.log | tail -20

# On the replica, check if the replication slot is failing
select * from pg_replication_slots;
```

## Root Cause

- **IKE/IPsec SA lifetime mismatch**: On-prem ASA Phase 2 lifetime (e.g., 8 hours) differs from AWS (e.g., 1 hour); one side's SA expires and drops it without coordinating the re-key, causing traffic loss until renegotiation
- **DPD threshold too aggressive**: DPD is enabled on one side but not mirrored on the other; transient jitter on the uplink triggers DPD timeout and drops the tunnel
- **Middlebox/NAT idle timeout**: An infrastructure device (ISP CGNAT, or on-prem WAN optimizer) has an idle timeout (~120s) that silently kills the ESP/UDP 500/4500 flow between rekeys; the tunnel stays "up" in theory but the path is dead until reconnect
- **MTU/fragmentation issues**: The path MTU is lower than expected; large packets (DB streaming with write-ahead-logging, or WAL batches) fragment and get dropped, causing TCP resets that appear as "drops"
- **IKE version negotiation**: one side keeps renegotiating IKEv1 vs IKEv2 incorrectly, or re-authenticating, causing periodic reconnects
- **Route flap**: A static route or BGP session over the tunnel is flapping, dropping the tunnel state from the routing table

## Immediate Mitigation

1. If a tunnel is down, force it to re-establish: `aws ec2 modify-vpn-tunnel-options` or on the ASA `clear crypto ipsec sa` / `clear crypto ikev1 sa`
2. Extend SA lifetime settings to an aligned value on both sides (e.g., 28800s) and enable aggressive dead-peer-detection so recovery is fast
3. If the middlebox is killing the tunnel, increase the SA lifetime / enable DPD keepalive so the path stays active
4. For database replication: increase WAL pressure tolerance (increase wal_keep_segments or archive at shorter intervals) so gaps stay small during tunnel re-negotiation
5. To restore replication after a drop quickly, run the WAL gap recovery (pg_receivewal / streaming resumes automatically when the tunnel is back)

## Permanent Fix

1. Align Phase 1 and Phase 2 lifetimes and DPD settings between the ASA and AWS (use the AWS-recommended VPN config values)
2. Enable Dead Peer Detection on both ends with consistent intervals
3. Ensure UDP 500/4500 NAT-T is properly opened and configured through any middlebox/NAT device (many providers route VPN via NAT; verify ESP or NAT-T)
4. Enforce the correct MTU (typically 1400 for GRE, or 8972 jumbo if supported) across the tunnel and disable fragmentation issues by setting the appropriate do-not-fragment behavior
5. Implement monitoring and alerts on VPN tunnel uptime and DPD failures — alert before the drop becomes visible to the DB
6. For DB replication resilience: use a replication slot plus `wal_receiver_timeout`/`wal_sender_timeout` smaller than the DPD detection so reconnects are clean, and monitor replication lag with alerts

## Monitoring

- Monitor VPN tunnel status (CloudWatch TunnelState) and alert on transitions
- Monitor IKE/SA re-key events and counters from the ASA logs
- Track replication lag between primary and replica (pg_stat_replication / pg_stat_wal_receiver)
- Set up alerts on DPD failures / re-key timeouts from the firewall
- Monitor the WAN path to the AWS endpoint (mtr / ping with jitter measurements)
- Track tunnel data throughput (TunnelDataIn/Out metrics) to detect degradation before drops
- Monitor PostgreSQL WAL gaps and deadlock/lock timeouts in the replication session

## Security

- Ensure IPsec SA encryption/hashing are at least AES-256 / SHA-256 when renegotiating; check for old IKEv1 with weak crypto
- Restrict the tunnel traffic to only the required subnets (source/destination in the crypto map) to reduce attack surface
- Ensure PSK/CA keys are rotated and stored securely (AWS VPN uses pre-shared keys or certs; store them in the secret manager)
- Enable logging of IPsec rejections and audit tunnel configuration changes
- Verify that tunnel re-negotiation doesn't expose credentials or use outdated key exchange algorithms

## Production Considerations

- **High Availability**: AWS provides 2 tunnels per VPN; use them for failover and load. Run the DB replication over the primary tunnel and have the fallback ready. The tunnel drop impact should be studied — post-recovery the DB gap recovery must not overload WAN
- **Scalability**: The VPN bandwidth should be sized for the DB replication stream (WAL throughput). Use the right VPN connection type (IPsec with appropriate throughput limits)
- **Reliability**: Monitor DPD and keepalive closely; a healthy tunnel with periodic drops undermines the DR setup. Configure the DB replication to tolerate WAL gaps and resume cleanly
- **Cost**: VPN tunnels are cheaper than dedicated circuits, but DR replication downtime is expensive. Invest in monitoring and aligned configs before expanding DR
- **Compliance**: Data replication is subject to residency/compliance (PCI/GDPR); the tunnel must enforce strong crypto and no data should transit any unencrypted path
- **Operational**: Document the tunnel parameters, both ends' configs, and a recovery runbook (how to re-establish the tunnel and how to recover DB replication quickly after a gap)

## Senior-Level Answer

I would check what "drop" means precisely: IKE Phase 1 loss vs. IPsec SA re-key. Then compare Phase 1/2 lifetimes and DPD settings between the ASA and AWS, test the WAN path to the AWS endpoint for jitter/loss right before the drops, and check for middlebox/NAT-T idle timeouts. The most common causes are lifetime mismatches (re-key on one side that the peer doesn't recognize) or an aggressive DPD killing the tunnel on transient jitter. Align the IPsec parameters using AWS's recommended config, increase DPD keepalives, and monitor replication lag with alerts to prevent the DB gaps from becoming permanent.

## Architect-Level Answer

Intermittent tunnel drops for DB replication indicate the DR link isn't meeting its reliability SLO. Architecturally, I would move the DR replication to a purpose-built, managed solution with built-in reliability guarantees — for PostgreSQL, AWS Database Migration Service (DMS) or Aurora Global Database (for Aurora) that offer managed replication with automatic recovery, eliminating dependence on the VPN for replication. For the VPN itself, I would adopt a redundancy plan: two independent VPN connections and BGP over the tunnels so routing automatically switches, or replace the site-to-site commercial link with AWS Direct Connect if the path MTU/reliability requires it. Above all, the DR replication should be designed for the link to drop: the WAL stream should be continuously archived/pushed locally and replayed after reconnection to tolerate any tunnel outage.

## Follow-Up Questions

1. How would you distinguish between an IPsec SA (Phase 2) re-key failure and a full IKE (Phase 1) timeout causing the drop?
2. If DPD is enabled on both sides with different intervals, what happens to the tunnel and how do you choose matching DPD settings?
3. What are the trade-offs of using AWS Direct Connect vs. a site-to-site VPN for database replication reliability and cost?
4. How would you recover replication efficiently if the tunnel stays down for 30 minutes and the WAL gap exceeds your local WAL archive capacity?
5. If both VPN tunnels drop simultaneously, what does that indicate about the failure domain, and how would your fix differ from a single-tunnel drop?