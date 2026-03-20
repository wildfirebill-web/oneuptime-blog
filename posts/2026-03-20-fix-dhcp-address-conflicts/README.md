# How to Fix DHCP Address Conflicts

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DHCP, Networking, Address Conflicts, Troubleshooting, sysadmin

Description: DHCP address conflicts occur when two devices use the same IP simultaneously, causing network failures that can be resolved by identifying the conflicting hosts, using DHCP conflict detection, and converting one device to a reservation or exclusion.

## What Causes DHCP Address Conflicts

1. A static IP was assigned to a host that overlaps with the DHCP pool.
2. DHCP server leased an IP that another device already had configured statically.
3. A stale lease was assigned to a new device while the original device was offline.
4. Multiple DHCP servers handed out the same address.

## ISC dhcpd: Built-in Conflict Detection

dhcpd can ping an address before offering it:

```
# /etc/dhcp/dhcpd.conf
# Ping an address before offering it to detect conflicts
ping-check true;
ping-timeout 1;
```

## Detecting Active Conflicts

```bash
# On Linux: check ARP table for duplicate IPs
arp -n | sort | uniq -w16 -d

# Scan subnet for duplicate IPs with nmap
nmap -sn 192.168.1.0/24 --open | grep -E "192\.168\.1\." > /tmp/hosts.txt
# Look for same IP appearing twice

# arping to identify which MAC has an IP
sudo arping -I eth0 192.168.1.50
```

## Finding Conflicted Leases

```bash
# ISC dhcpd: look for conflict entries in lease file
grep -i "conflict\|decline" /var/lib/dhcp/dhcpd.leases

# Also check DHCP server logs
journalctl -u isc-dhcp-server | grep -i "DHCPDECLINE\|conflict"
```

## Resolving Conflicts

### Step 1: Identify Both Devices

```bash
# Find the MAC address for each IP claiming the address
sudo arping -D -I eth0 192.168.1.50
# -D = duplicate address detection mode; exits with non-zero if duplicate detected

# Find which devices have a given IP
sudo arp-scan --interface=eth0 192.168.1.50
```

### Step 2: Remove Stale Leases

```bash
# Edit dhcpd.leases to remove the conflicted lease
sudo systemctl stop isc-dhcp-server
sudo vi /var/lib/dhcp/dhcpd.leases
# Remove or comment out the conflicted lease block
sudo systemctl start isc-dhcp-server
```

### Step 3: Exclude the Static IP from the DHCP Pool

```
# Add to /etc/dhcp/dhcpd.conf
# Exclude addresses used by static devices from the dynamic range
subnet 192.168.1.0 netmask 255.255.255.0 {
    # Dynamic pool starts AFTER static device range
    range 192.168.1.50 192.168.1.200;
    # OR convert the static device to a reservation
}
```

## Windows Server: View and Resolve Conflicts

```powershell
# View conflicted leases
Get-DhcpServerv4Conflict -ScopeId 192.168.1.0

# Remove a conflicted record
Remove-DhcpServerv4Conflict -ScopeId 192.168.1.0 -IPAddress 192.168.1.50

# Enable conflict detection (number of pings before offering)
Set-DhcpServerv4Scope -ScopeId 192.168.1.0 -ConflictDetectionAttempts 1
```

## Key Takeaways

- Conflicts most commonly occur when static IPs overlap with the DHCP pool.
- Enable conflict detection (`ping-check true` in dhcpd) to prevent re-offering in-use addresses.
- Use `arping -D` to detect duplicate addresses on the network.
- Best practice: keep static IP ranges (1–49) completely separate from the DHCP pool (50–254).
