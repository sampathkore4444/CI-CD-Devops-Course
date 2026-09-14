# 5. Cannot SSH into Production Server

## Scenario
It's 8:15 AM on a Monday morning. You need to perform an urgent security patch on prod-web-01. You try to SSH in and get "Connection timed out." You try from a different jump host — same result. A colleague tries — same result. However, the server is responding to pings, and the application it hosts is still serving traffic normally through the load balancer. The server runs nginx as a reverse proxy and has been up for 437 days without a reboot. You need to regain SSH access to apply the patch, but you can't afford to disrupt the running application.

## Interviewer Question
"You cannot SSH into a critical production server. The server still responds to pings and serves traffic, but SSH is unreachable. How do you regain access without disrupting the running application?"

## What I Should Think About
- **SSH failure modes**: sshd down, firewall blocking, port 22 blocked, max sessions reached, disk full, PAM issues, key-based auth issues
- **Out-of-band access**: IPMI/iLO/iDRAC, cloud console, serial console
- **Multiple NICs**: Can you reach the server on a different network interface?
- **Firewall**: Did a recent rule change block SSH?
- **sshd config**: Did someone modify sshd_config?
- **Disk full**: Can sshd write its logs? Does it have room for new sessions?
- **Process limits**: Has sshd hit MaxSessions or MaxStartups?
- **Last resort**: Can you reboot without downtime? HA failover?

## Ideal Answer

**Phase 1: Quick Diagnostics from Outside**

First, confirm the SSH port is actually unreachable vs. the server being down:

```bash
# From a jump host
ping -c 3 prod-web-01
telnet prod-web-01 22
nmap -p 22 prod-web-01
nc -zv prod-web-01 22
```

If telnet/nmap shows port 22 is closed or filtered, it's a firewall or sshd issue. If it shows "connection refused," sshd isn't running. If it hangs, it could be a firewall drop rule.

**Phase 2: Try Alternative Access Methods**

```bash
# Try SSH on other ports (if configured)
ssh -p 2222 prod-web-01

# Try a different network interface
ssh admin@10.0.1.100  # Internal NIC
ssh admin@172.16.0.100  # Docker network NIC

# Try the cloud console (if AWS)
aws ec2 instance-status --instance-ids i-0abc123def456
aws ec2 get-console-output --instance-id i-0abc123def456

# Try via IPMI/iLO/iDRAC (bare metal)
ipmitool -I lanplus -H 192.168.1.100 -U admin -P password sol activate
```

**Phase 3: Use the Cloud/BMC Console**

If the server is an EC2 instance:
```bash
# Get the system log to see what happened
aws ec2 get-console-output --instance-id i-0abc123 --output text | tail -50

# Request a serial console
aws ec2-instance-connect send-serial-console-ssh-public-key \
  --instance-id i-0abc123 \
  --serial-port 0 \
  --ssh-public-key file://my_key.pub
```

If bare metal with IPMI:
```bash
ipmitool -I lanplus -H BMC_IP -U admin -P password power status
ipmitool -I lanplus -H BMC_IP -U admin -P password sol activate
```

**Phase 4: Remote Fix via Serial Console or Live Recovery**

Once you have console access:
```bash
# Check if sshd is running
systemctl status sshd
ps aux | grep sshd

# Check firewall rules
iptables -L -n | grep -E "22|ssh"
firewall-cmd --list-all 2>/dev/null

# Check disk space (if full, sshd can't accept new sessions)
df -h

# Check sshd config
sshd -T | grep -E "port|maxsessions|maxstartups|permitrootlogin"

# Check if MaxSessions/MaxStartups is hit
ss -s | grep -i listen
netstat -tlnp | grep :22

# Check auth logs
tail -50 /var/log/auth.log  # or /var/log/secure
```

**Phase 5: Common Fixes**

```bash
# If sshd is down
systemctl start sshd

# If disk is full
# Clean up space quickly
journalctl --vacuum-size=100M
find /var/log -name "*.gz" -mtime +7 -delete

# If firewall is blocking
iptables -I INPUT -p tcp --dport 22 -j ACCEPT

# If MaxStartups is hit
# Kill idle SSH sessions
ss -tnp | grep ssh | head -20

# Restart sshd after fixing config
systemctl restart sshd
```

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│                  Network                                 │
│                                                          │
│  ┌──────────┐    ┌──────────┐    ┌──────────────────┐   │
│  │ Your     │───▶│ Load     │───▶│ prod-web-01      │   │
│  │ Laptop   │    │ Balancer │    │                  │   │
│  └──────────┘    └──────────┘    │ Port 80:  ✓ UP   │   │
│       │                          │ Port 22:  ✗ DOWN │   │
│       │                          │ Ping:     ✓ OK   │   │
│       ▼                          │                  │   │
│  ┌──────────┐                    │ SSH: timeout     │   │
│  │ Jump Host│──────X───────────▶│                  │   │
│  │ (SSH)    │                    └────────┬─────────┘   │
│  └──────────┘                             │             │
│                                    ┌──────▼──────┐      │
│                                    │ IPMI/BMC    │      │
│                                    │ (out-of-band│      │
│                                    │  access)    │      │
│                                    └─────────────┘      │
└──────────────────────────────────────────────────────────┘
```

## Investigation

1. **Test SSH connectivity**: Run `nc -zv prod-web-01 22` and `nmap -p 22 prod-web-01` to determine if port 22 is open, closed, or filtered.
2. **Check if the server is reachable**: Run `ping prod-web-01` and `traceroute prod-web-01` to confirm network connectivity.
3. **Try alternative SSH ports**: If the server has SSH on a non-standard port, try that: `ssh -p 2222 prod-web-01`.
4. **Check different network interfaces**: The server may have multiple NICs; try each IP address.
5. **Attempt cloud console access**: Use AWS EC2 Serial Console, GCP Serial Port, or Azure Serial Console.
6. **Attempt BMC/IPMI access**: Use out-of-band management to get a serial console.
7. **Once on console, check sshd status**: `systemctl status sshd` and check for configuration errors.
8. **Check firewall rules**: `iptables -L -n` and `firewall-cmd --list-all`.
9. **Check disk space**: If the disk is full, sshd may refuse connections.
10. **Check auth logs**: `tail -50 /var/log/auth.log` for clues about connection attempts.

## Commands

```bash
# 1. Test SSH port connectivity
nc -zv prod-web-01 22
nmap -p 22 prod-web-01
telnet prod-web-01 22

# 2. Check alternative ports
nmap -p 22,2222,8022 prod-web-01

# 3. Try different network interfaces
ip route get <server-ip>  # Find which route to use
ssh admin@<server-secondary-ip>

# 4. AWS: Get console output
aws ec2 get-console-output --instance-id i-0abc123 --output text | tail -100

# 5. AWS: Send serial console SSH key
aws ec2-instance-connect send-serial-console-ssh-public-key \
  --instance-id i-0abc123 \
  --serial-port 0 \
  --ssh-public-key file://~/.ssh/id_rsa.pub

# 6. IPMI: Activate serial console
ipmitool -I lanplus -H BMC_IP -U admin -P password sol activate

# 7. Once on console - check sshd
systemctl status sshd
sshd -t  # Test config syntax
sshd -T | grep -i port

# 8. Check firewall
iptables -L -n -v | head -30
firewall-cmd --list-all 2>/dev/null
ufw status 2>/dev/null

# 9. Check disk space
df -h / /var /tmp

# 10. Check auth logs
tail -50 /var/log/auth.log
tail -50 /var/log/secure

# 11. Check for brute-force protection blocking you
fail2ban-client status sshd 2>/dev/null
grep "too many" /var/log/auth.log | tail -5

# 12. Check sshd process and connections
ps aux | grep sshd
ss -tnp | grep sshd | wc -l  # Count active connections
ss -s | grep -i listen

# 13. Check PAM configuration
cat /etc/pam.d/sshd | grep -v "^#"

# 14. Quick fix: restart sshd
systemctl restart sshd

# 15. Quick fix: allow SSH through firewall
iptables -I INPUT -p tcp --dport 22 -j ACCEPT
```

## Root Cause

| Root Cause | Detection | Resolution |
|---|---|---|
| sshd service crashed or stopped | `systemctl status sshd` shows inactive | `systemctl start sshd`, investigate why it stopped |
| Firewall rule blocking SSH | `iptables -L` shows DROP/REJECT on port 22 | Add ACCEPT rule for port 22 |
| Disk full preventing new sessions | `df -h` shows 100% usage | Free disk space, restart sshd |
| MaxStartups/MaxSessions reached | `ss -tnp \| grep sshd` shows many connections | Kill idle sessions, increase limits in sshd_config |
| fail2ban blocking your IP | `fail2ban-client status sshd` shows your IP | Unban your IP: `fail2ban-client set sshd unbanip YOUR_IP` |
| sshd_config syntax error | `sshd -t` shows error | Fix the config, restart sshd |
| PAM misconfiguration | Auth logs show PAM errors | Fix /etc/pam.d/sshd |
| Key-based auth failure | Logs show "publickey denied" | Check authorized_keys, permissions |
| Host key missing or corrupted | sshd fails to start, logs show host key error | Regenerate host keys: `ssh-keygen -A` |

## Immediate Mitigation

1. **Try cloud/BMC console** immediately — this is the fastest path to regaining access.
2. **Check from another network path** — try SSH via a different NIC or VPN.
3. **If fail2ban is blocking you**: `fail2ban-client set sshd unbanip YOUR_IP`.
4. **If disk is full**: Use console access to clean up space, then restart sshd.
5. **If sshd is stopped**: Start it from console: `systemctl start sshd`.
6. **If firewall is blocking**: From console: `iptables -I INPUT -p tcp --dport 22 -j ACCEPT`.
7. **Last resort**: If you can't fix it and need the patch urgently, coordinate an HA failover to another web server and schedule a maintenance window for this one.

## Permanent Fix

1. **Implement out-of-band access**: Ensure every production server has IPMI/BMC or cloud serial console access configured and tested.
2. **Set up SSH monitoring**: Alert when sshd stops or port 22 becomes unreachable.
3. **Configure fail2ban whitelist**: Add your management IPs to the fail2ban ignore list.
4. **Implement connection limits**: Set `MaxStartups 10:30:60` and `MaxSessions 10` in sshd_config.
5. **Disk monitoring**: Ensure disk usage alerts trigger before 100%.
6. **SSH bastion hosts**: Use a hardened bastion host for all production SSH access.
7. **Configuration management**: Use Ansible/Salt to manage sshd_config across all servers.

## Monitoring

- **SSH port monitoring**: Check port 22 is accessible from the jump host every minute.
- **sshd process monitoring**: Alert if the sshd process is not running.
- **Active SSH sessions**: Alert when session count approaches MaxSessions.
- **fail2ban bans**: Monitor for excessive bans that could indicate legitimate access being blocked.
- **Disk usage**: Alert at 85% to prevent disk-related SSH failures.
- **Firewall changes**: Audit and alert on any iptables/firewalld rule changes.

## Security

- **Out-of-band access security**: BMC/IPMI interfaces should be on a separate management network with strict access controls.
- **Bastion host hardening**: The bastion host should be minimal, hardened, and have enhanced logging.
- **SSH key management**: Use centralized SSH key management with proper rotation.
- **Audit all access**: Log all SSH connections and commands for compliance.
- **Emergency access procedures**: Document and test emergency access procedures regularly.

## Production Considerations

- **HA**: This server is behind a load balancer. If SSH is down but the app works, other servers can handle traffic while this one is fixed.
- **Zero-downtime patching**: If this server can't be accessed, can the patch be applied through the load balancer's traffic routing?
- **Blast radius**: One server losing SSH access shouldn't affect the overall service. If it does, the architecture needs improvement.
- **Documentation**: Maintain an up-to-date runbook for SSH access recovery.
- **Regular testing**: Test out-of-band access methods quarterly to ensure they work.

## Senior-Level Answer

"I'd start by confirming the SSH port is unreachable with `nc -zv` and `nmap`, while verifying the server is otherwise healthy via ping and application health checks. I'd immediately attempt out-of-band access via cloud serial console or IPMI. Once on the console, I'd check `systemctl status sshd`, firewall rules, disk space, and auth logs. Common fixes include restarting sshd, whitelisting my IP in fail2ban, or freeing disk space. For prevention, I'd ensure all production servers have tested out-of-band access, implement SSH port monitoring, configure fail2ban whitelists for management IPs, and set up a bastion host architecture for all production SSH access."

## Architect-Level Answer

"SSH access failure on a production server reveals gaps in three areas: access architecture, monitoring, and disaster recovery. First, we need a proper bastion host architecture where all production SSH access flows through hardened, monitored jump servers with proper access controls. Second, every server should have out-of-band access (cloud serial console or IPMI) that's regularly tested — I'd add quarterly drills to the operational calendar. Third, we need proactive monitoring: SSH port checks, sshd process monitoring, and disk usage alerts that prevent the most common causes of SSH failure. For the long term, I'd evaluate whether we should move to certificate-based SSH with short-lived credentials managed by a central authority, eliminating the key distribution problem entirely. I'd also recommend implementing automated remediation: if sshd goes down, an automated runbook could restart it via cloud-init or BMC without human intervention."

## Follow-Up Questions

1. "What's the difference between sshd's `MaxStartups`, `MaxSessions`, and `MaxAuthTries`? How would you tune each?"
2. "How does fail2ban work internally? What would you do if fail2ban itself crashed and left the firewall in a bad state?"
3. "Explain how SSH tunneling works. How would you use it to access a server behind a NAT?"
4. "How would you set up a zero-trust SSH access model for a fleet of 500 production servers?"
5. "What's the difference between using a jump host proxy with `ProxyJump` versus `ProxyCommand`? When would you use each?"
