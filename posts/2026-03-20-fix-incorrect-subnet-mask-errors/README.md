# How to Fix Incorrect Subnet Mask Configuration Errors

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Subnet Mask, CIDR, Misconfiguration, Troubleshooting, Network

Description: Learn how to detect and fix incorrect subnet mask configurations that cause devices to believe they're in different subnets, preventing communication even when hosts are on the same physical network.

## How Subnet Mask Errors Cause Problems

A subnet mask mismatch means two devices on the same physical network think they're in different networks:

```text
Device A: IP 192.168.1.10, mask 255.255.255.0  → subnet 192.168.1.0/24
Device B: IP 192.168.1.20, mask 255.255.0.0    → subnet 192.168.0.0/16

Device A thinks B is remote (sends to gateway)
Device B thinks A is local (sends directly via ARP)
Result: asymmetric routing, no communication
```

## Step 1: Identify Misconfigured Devices

```bash
# Scan network and compare subnet masks

nmap -sn 192.168.1.0/24 -v

# On each device, check the mask
# Linux
ip addr show eth0
# or
ifconfig eth0

# Windows
ipconfig /all | findstr "Subnet Mask"
# Should show: 255.255.255.0 for a /24 network

# Cisco router/switch
show interfaces GigabitEthernet0/0
# Shows: Internet address is 192.168.1.1/24
```

## Step 2: Diagnose the Communication Failure

```bash
# Test if two devices on same network can reach each other
ping 192.168.1.20

# Check ARP to see if hosts are resolving each other
arp -n | grep 192.168.1.20

# If ping fails but they're on same switch:
# 1. Check if ARP is resolving (same MAC should appear)
# 2. If ARP resolves but ping fails: firewall issue
# 3. If ARP doesn't resolve: routing to gateway instead of direct

# Trace what happens to packets
ip route get 192.168.1.20
# Correct: shows "dev eth0 src 192.168.1.10" (direct)
# Wrong:   shows "via 192.168.1.1 dev eth0" (going through gateway)
```

## Step 3: Fix on Linux

```bash
# Temporary fix - set correct mask
sudo ip addr del 192.168.1.10/16 dev eth0    # Remove wrong mask
sudo ip addr add 192.168.1.10/24 dev eth0    # Add correct mask
sudo ip route add default via 192.168.1.1    # Restore default route

# Verify
ip addr show eth0
ip route show
```

```yaml
# Permanent fix via netplan (/etc/netplan/01-netcfg.yaml)
network:
  version: 2
  ethernets:
    eth0:
      addresses:
        - 192.168.1.10/24    # Correct CIDR notation
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8]
```

```bash
# NetworkManager
nmcli con mod "Wired connection 1" ipv4.addresses "192.168.1.10/24"
nmcli con mod "Wired connection 1" ipv4.gateway "192.168.1.1"
nmcli con up "Wired connection 1"
```

## Step 4: Fix on Windows

```powershell
# Check current config
Get-NetIPAddress -InterfaceAlias "Ethernet" | Select-Object IPAddress, PrefixLength

# Fix incorrect prefix length
# Remove bad address first
Remove-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 192.168.1.10 -Confirm:$false

# Add with correct prefix length (24 = 255.255.255.0)
New-NetIPAddress -InterfaceAlias "Ethernet" `
    -IPAddress 192.168.1.10 `
    -PrefixLength 24 `
    -DefaultGateway 192.168.1.1
```

```cmd
REM netsh alternative
netsh interface ip set address name="Ethernet" static 192.168.1.10 255.255.255.0 192.168.1.1
```

## Step 5: Fix on Cisco IOS

```text
Router# configure terminal
Router(config)# interface GigabitEthernet0/0
Router(config-if)# no ip address
Router(config-if)# ip address 192.168.1.1 255.255.255.0   ! /24 correct mask
Router(config-if)# no shutdown
Router(config-if)# end
Router# show interfaces GigabitEthernet0/0
Router# write memory
```

## Step 6: Audit All Devices for Mask Consistency

```python
#!/usr/bin/env python3
"""Audit subnet masks across devices"""
from ipaddress import ip_interface

expected_prefix = 24  # Expected /24 everywhere

devices = [
    ("router", "192.168.1.1/24"),
    ("server1", "192.168.1.10/24"),
    ("workstation", "192.168.1.20/16"),   # WRONG
    ("printer", "192.168.1.30/24"),
]

for name, addr_str in devices:
    iface = ip_interface(addr_str)
    if iface.network.prefixlen != expected_prefix:
        print(f"MISMATCH: {name} has /{iface.network.prefixlen}, expected /{expected_prefix}")
    else:
        print(f"OK: {name} {addr_str}")
```

## Conclusion

Subnet mask mismatches cause two devices to see each other as being in different networks, even when physically adjacent. Diagnose with `ip route get [target-ip]` - if it shows the gateway instead of direct, the local mask is wrong. Fix on Linux with `ip addr del/add` or netplan, on Windows with `New-NetIPAddress -PrefixLength 24`, and on Cisco with `ip address X.X.X.X 255.255.255.0`. Audit all devices systematically to ensure consistent masks across the network.
