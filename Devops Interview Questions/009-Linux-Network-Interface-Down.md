# 9. Network Interface Down on Production Server

## Scenario
At 1:15 PM on a Monday, the monitoring system alerts: "Network unreachable: prod-db-master-01 (10.0.1.50)." The server is the primary PostgreSQL database for the e-commerce platform. It has three network interfaces: `eth0` (10.0.1.50) for the application network, `eth1` (10.0.2.50) for the replication network, and `eth2` (192.168.1.50) for the management network. The application network interface (`eth0`) has gone down. The application is failing because it can't reach the database. However, you can still reach the server via the management network (`eth2`) using a jump host. You need to restore network connectivity without disrupting the database service.

## Interviewer Question
"One of the network interfaces on a production server has gone down. The server has multiple NICs for different network segments. Traffic is being affected. How do you diagnose and restore connectivity?"

## What I Should Think About
- **Interface state**: Is it admin down, carrier down, or link down?
- **Physical layer**: Cable, switch port, SFP, NIC hardware
- **Configuration**: IP, subnet, gateway, routes
- **Bonding/teaming**: Is this a bonded interface?
- **Switch side**: VLAN, port security, STP
- **Impact**: What traffic is affected? Can we route around it?
- **Multiple paths**: Can we use a different interface temporarily?

## Ideal Answer

**Phase 1: Assess the Interface State**

```bash
# Check interface status
ip link show
ip addr show

# Check detailed interface info
ethtool eth0
ethtool -S eth0

# Check if the interface is administratively down
ip link show eth0 | grep -E "state|mtu"

# Check dmesg for hardware/driver messages
dmesg | grep -i eth0 | tail -20

# Check routing table
ip route show
ip route get 10.0.1.1
```

The output shows:
```
3: eth0: <BROADCAST,MULTICAST> mtu 1500 qdisc pfifo_fast state DOWN
    link/ether 00:1a:2b:3c:4d:5e brd ff:ff:ff:ff:ff:ff
```

The interface is in DOWN state (not UP), which means either it's administratively down or the carrier is lost.

**Phase 2: Determine if it's Physical or Logical**

```bash
# Check if it's administratively down
ip link show eth0 | grep -o "state [A-Z]*"

# Check ethtool for carrier status
ethtool eth0 | grep -E "Link detected|Speed|Duplex"

# Check if the driver is loaded
lspci | grep -i ethernet
lsmod | grep <driver_name>

# Check for hardware errors
dmesg | grep -iE "eth0|NIC|link|carrier|error"
journalctl -k | grep -iE "eth0|link|carrier"

# Check if the cable is detected
ethtool eth0 | grep "Link detected"
```

**Phase 3: Try to Bring It Up**

```bash
# If administratively down
ip link set eth0 up

# If it comes up but no IP
ip addr add 10.0.1.50/24 dev eth0
ip route add default via 10.0.1.1 dev eth0

# If it's a carrier issue
# Try renegotiating link
ethtool -r eth0

# Check if the switch port is the issue
# (Requires switch access)
```

**Phase 4: If the Physical Interface is Dead**

```bash
# Check if the NIC is detected by the system
lspci | grep -i ethernet
lspci -vvv | grep -A 10 -i ethernet

# Check if the driver needs reloading
modprobe -r <driver_name>
modprobe <driver_name>

# If the NIC is physically damaged, use a different interface
# eth1 is on the replication network, but we can temporarily
# add the application network IP to it
ip addr add 10.0.1.50/24 dev eth1
ip route add 10.0.1.0/24 dev eth1

# Or use the management interface
ip addr add 10.0.1.50/24 dev eth2
```

**Phase 5: Permanent Fix**

```bash
# If it was a driver issue
echo "blacklist <bad_driver>" >> /etc/modprobe.d/blacklist.conf
echo "options <good_driver> param=value" >> /etc/modprobe.d/<driver>.conf

# If it was a cable/switch issue
# Replace the cable or fix the switch port

# Set up network bonding for redundancy
# /etc/netplan/01-network.yaml (Ubuntu)
network:
  version: 2
  ethernets:
    bond0:
      interfaces:
        - eth0
        - eth1
      addresses:
        - 10.0.1.50/24
      gateway4: 10.0.1.1
      parameters:
        mode: active-backup
        primary: eth0
```

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│               prod-db-master-01                          │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │ eth0 (10.0.1.50) ← Application Network            │  │
│  │ State: DOWN ← PROBLEM                              │  │
│  │ Cable → Switch Port 24 → To App Servers            │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │ eth1 (10.0.2.50) ← Replication Network            │  │
│  │ State: UP ✓                                        │  │
│  │ Cable → Switch Port 25 → To DB Replicas            │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │ eth2 (192.168.1.50) ← Management Network          │  │
│  │ State: UP ✓                                        │  │
│  │ Cable → Management Switch → To Jump Hosts          │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  PostgreSQL: Running (can be managed via eth2)           │
│  Impact: App → DB connection timeout                     │
└──────────────────────────────────────────────────────────┘

     Switch (prod-sw-03)
     ┌─────────────────────────────────┐
     │ Port 24: eth0 ← LINK DOWN      │
     │ Port 25: eth1 ← LINK UP        │
     │ Port 30: eth2 ← LINK UP        │
     └─────────────────────────────────┘
```

## Investigation

1. **Check interface state**: Run `ip link show` to see if the interface is UP or DOWN.
2. **Check if it's administrative**: Run `ip link show eth0` — if state is DOWN, it was set down manually. If state is NO-CARRIER, the physical link is lost.
3. **Check physical layer**: Run `ethtool eth0` to see if the cable is detected (Link detected: yes/no), speed, and duplex.
4. **Check hardware/driver**: Run `dmesg | grep eth0` and `lspci | grep -i ethernet` to check for hardware issues.
5. **Check switch side**: Verify the switch port is active, correct VLAN, and not err-disabled.
6. **Check routing**: Run `ip route show` to see if routes through eth0 still exist.
7. **Check for recent changes**: Review if anyone made network changes before the outage.
8. **Test connectivity from the server**: `ping -I eth1 10.0.2.1` to confirm other interfaces work.

## Commands

```bash
# 1. Check all interfaces
ip link show
ip addr show

# 2. Check specific interface details
ip -s link show eth0
ethtool eth0
ethtool -S eth0  # NIC statistics

# 3. Check if interface is admin down
ip link show eth0 | grep state

# 4. Bring interface up
ip link set eth0 up

# 5. Add IP address if missing
ip addr add 10.0.1.50/24 dev eth0

# 6. Add default route if missing
ip route add default via 10.0.1.1 dev eth0

# 7. Check driver and firmware
ethtool -i eth0
lspci -vvv | grep -A 20 -i ethernet

# 8. Reload NIC driver
modprobe -r e1000e  # Remove
modprobe e1000e    # Reload

# 9. Reset the NIC
ethtool -r eth0

# 10. Check for link issues
ethtool eth0 | grep -E "Speed|Duplex|Link detected|Auto-negotiation"

# 11. Check routing table
ip route show
ip route get 10.0.1.1

# 12. Check ARP table
ip neigh show
arp -n

# 13. Monitor interface status
watch -n 2 "ip link show eth0"

# 14. Check systemd-networkd status (if used)
systemctl status systemd-networkd
networkctl status eth0

# 15. Check if NetworkManager is interfering
nmcli device status 2>/dev/null
nmcli device show eth0 2>/dev/null
```

## Root Cause

| Root Cause | Detection | Resolution |
|---|---|---|
| Cable unplugged or damaged | `ethtool` shows "Link detected: no" | Reseat or replace cable |
| Switch port err-disabled | Switch shows port in err-disabled state | Fix cause, `errdisable recovery` on switch |
| NIC driver crash | `dmesg` shows driver errors, NIC not in `lspci` | Reload driver with `modprobe -r && modprobe` |
| NIC hardware failure | `lspci` doesn't show NIC, driver reload fails | Add IP to another interface, replace NIC |
| Interface admin down | `ip link show` shows state DOWN | `ip link set eth0 up` |
| VLAN misconfiguration | Traffic not reaching the interface | Fix VLAN configuration on switch and/or server |
| STP blocking | Switch port in blocking state | Enable portfast on access ports |

## Immediate Mitigation

1. **Bring the interface up**: `ip link set eth0 up` — if it was administratively down.
2. **Add IP if missing**: `ip addr add 10.0.1.50/24 dev eth0 && ip route add default via 10.0.1.1 dev eth0`.
3. **If physical issue**: Temporarily add the application network IP to another interface: `ip addr add 10.0.1.50/24 dev eth1`.
4. **Reload the driver**: `modprobe -r <driver> && modprobe <driver>`.
5. **If the NIC is dead**: Route application traffic through the management network as a temporary workaround.
6. **Notify the team**: Communicate that the database is reachable via an alternate path while the NIC is being fixed.

## Permanent Fix

1. **Implement NIC bonding**: Use active-backup bonding so a single NIC failure doesn't cause an outage.
2. **Set up network monitoring**: Alert when an interface goes down or packet loss exceeds thresholds.
3. **Enable NIC watchdog**: Configure `NICWatchdogTimer` in BIOS/BMC.
4. **Document the network layout**: Maintain up-to-date network diagrams.
5. **Implement auto-recovery**: Use `systemd-networkd` or `NetworkManager` to auto-restart interfaces on failure.

## Monitoring

- **Interface status**: Alert immediately when any interface goes DOWN.
- **Packet loss**: Monitor per-interface packet loss and errors.
- **Bandwidth utilization**: Alert when interface utilization exceeds 80%.
- **CRC errors**: Monitor NIC statistics for CRC errors indicating hardware issues.
- **Switch port status**: Monitor switch port status for the corresponding port.

## Security

- **Network isolation**: Ensure management network (eth2) is on a separate VLAN with strict access controls.
- **MAC address filtering**: If using port security, ensure new MAC addresses aren't blocked after NIC replacement.
- **Traffic routing**: Temporarily routing app traffic through the management network reduces network segmentation.

## Production Considerations

- **HA**: The database has replicas. If the master is unreachable, ensure the application can failover to a replica.
- **Data consistency**: If the master becomes unreachable during writes, check for replication lag and data consistency.
- **NIC bonding**: Active-backup bonding provides redundancy without switch configuration changes.
- **Multiple NICs**: The multi-NIC design saved us here — the management network allowed remote access.
- **Change management**: Document the network changes made during the incident.

## Senior-Level Answer

"I'd check the interface state with `ip link show` and `ethtool eth0` to determine if it's a physical or logical issue. If it's administratively down, `ip link set eth0 up`. If the physical link is down, I'd check the cable, switch port, and NIC driver. For immediate recovery, I'd add the application IP to an alternate interface (`ip addr add 10.0.1.50/24 dev eth1`). For permanent prevention, I'd implement NIC bonding in active-backup mode so a single NIC failure doesn't cause an outage, and set up interface-level monitoring with immediate alerts."

## Architect-Level Answer

"This incident reveals a single point of failure in our network architecture. A single NIC failure shouldn't cause an application outage. I'd implement three layers of network redundancy: First, NIC bonding (LACP or active-backup) on every server for interface-level redundancy. Second, multiple network paths with diverse physical routing for link-level redundancy. Third, the application should support database connection failover to replicas automatically. I'd also recommend implementing a network health monitoring system that includes interface status, switch port status, and network path verification. For the database specifically, I'd evaluate whether it should use a virtual IP (via keepalived or similar) that can float between servers, eliminating the dependency on a single server's network interface."

## Follow-Up Questions

1. "Explain the difference between LACP (802.3ad), active-backup, and balance-rr bonding modes. When would you use each?"
2. "What is STP (Spanning Tree Protocol) and how can it cause a network interface to appear down?"
3. "How does a virtual IP (VIP) work with keepalived? How would you use it for database high availability?"
4. "What's the difference between a VLAN and a separate physical network? When would you choose one over the other?"
5. "Design a network architecture for a database cluster that provides zero-downtime network failover."
