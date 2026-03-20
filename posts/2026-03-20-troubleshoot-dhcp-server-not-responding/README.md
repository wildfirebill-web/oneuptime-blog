# How to Troubleshoot DHCP Server Not Responding

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DHCP, Troubleshooting, Networking, Network Diagnostics, sysadmin

Description: When a DHCP server stops responding, clients fall back to APIPA (169.254.x.x) addresses, and systematic diagnosis involves checking service status, firewall rules, interface bindings, and pool availability.

## Symptoms

- Clients receive 169.254.x.x (APIPA) addresses
- `ipconfig /release && ipconfig /renew` (Windows) or `dhclient -v eth0` (Linux) fails
- Connections drop when moving to a new network

## Step 1: Check DHCP Service Status

```bash
# ISC dhcpd
sudo systemctl status isc-dhcp-server
journalctl -u isc-dhcp-server -n 50

# dnsmasq
sudo systemctl status dnsmasq
journalctl -u dnsmasq -n 50

# Windows Server
Get-Service -Name "DHCPServer"
Get-EventLog -LogName System -Source DhcpServer -Newest 20
```

## Step 2: Verify the DHCP Server Process is Listening

```bash
# Should show UDP port 67 listening
ss -ulnp | grep 67

# Or
netstat -ulnp | grep dhcpd
```

## Step 3: Check Firewall Rules

```bash
# Linux: ensure UDP 67 is not blocked
sudo iptables -L INPUT -n | grep 67
sudo iptables -L -n | grep -E "DHCP|67|68"

# Allow DHCP if blocked
sudo iptables -A INPUT -p udp --dport 67 -j ACCEPT
sudo iptables -A INPUT -p udp --dport 68 -j ACCEPT

# Windows Server firewall
Get-NetFirewallRule -DisplayName "*DHCP*"
```

## Step 4: Verify Interface Binding

The DHCP server must be bound to the correct interface:

```bash
# ISC dhcpd: check /etc/default/isc-dhcp-server
grep INTERFACES /etc/default/isc-dhcp-server

# Check if the interface has an IP in the declared subnet
ip addr show eth0

# dnsmasq: check interface in /etc/dnsmasq.conf
grep ^interface /etc/dnsmasq.conf
```

## Step 5: Check Address Pool Exhaustion

```bash
# Count active leases vs pool size
grep "^lease" /var/lib/dhcp/dhcpd.leases | wc -l

# View all currently bound leases
grep -A5 "binding state active" /var/lib/dhcp/dhcpd.leases

# Windows Server
Get-DhcpServerv4ScopeStatistics -ScopeId 192.168.1.0
```

## Step 6: Test with a Manual DHCP Request

```bash
# Send a DHCP discover and see if server responds
sudo nmap --script broadcast-dhcp-discover -e eth0

# Or use dhcping
sudo dhcping -s 192.168.1.1 -h AA:BB:CC:DD:EE:FF
```

## Step 7: Check for Rogue DHCP Servers

```bash
# Multiple DHCP servers can conflict
sudo nmap --script broadcast-dhcp-discover -e eth0
# If more than one DHCP offer is received, there's a rogue server
```

## Key Takeaways

- Check service status and logs first (`systemctl status`, `journalctl`).
- Verify UDP port 67 is not blocked by the firewall.
- Check that the DHCP server is bound to the correct interface.
- Pool exhaustion (no free addresses) causes the same symptoms as an unresponsive server.
