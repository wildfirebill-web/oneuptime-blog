# How to Troubleshoot IPv6 in Virtual Machine Environments

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Troubleshooting, Virtual Machine, KVM, VMware, NDP, SLAAC

Description: Diagnose and resolve common IPv6 connectivity problems in virtual machine environments, covering address assignment failures, NDP issues, routing problems, and hypervisor-specific configurations.

## Introduction

IPv6 connectivity problems in virtual machine environments arise from multiple layers: the hypervisor virtual switch, the host kernel's bridge/forwarding configuration, the guest OS network stack, and the physical network. A systematic diagnostic approach - starting from address assignment and working up the stack - identifies the failure point quickly.

## Common IPv6 VM Issues

```nginx
Problem Categories:
│
├─ No IPv6 address assigned
│  ├─ SLAAC: no Router Advertisement received
│  ├─ DHCPv6: no response from DHCPv6 server
│  └─ Static: cloud-init/configuration not applied
│
├─ Address assigned but no connectivity
│  ├─ NDP failure (can't resolve gateway MAC)
│  ├─ Route missing (no default route)
│  └─ Firewall blocking IPv6 (host or network)
│
├─ IPv6 works within subnet but not external
│  ├─ No IPv6 default route
│  ├─ Host IPv6 forwarding disabled
│  └─ Upstream router not propagating IPv6
│
└─ Intermittent failures
   ├─ MTU mismatch (PMTUD broken)
   └─ NDP cache timeouts causing temporary outages
```

## Step 1: Verify IPv6 Address Assignment

```bash
# Inside the VM: check IPv6 addresses

ip -6 addr show

# Expected output for a working VM:
# 2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP>
#     inet6 2001:db8::100/64 scope global dynamic noprefixroute
#        valid_lft 86299sec preferred_lft 14299sec
#     inet6 fe80::5054:ff:feab:cd01/64 scope link

# If only link-local (fe80::) - SLAAC/DHCPv6 not working:
# Check if Router Advertisements are being received
tcpdump -i eth0 -n -c5 "icmp6 and icmp6[0] == 134"
# Type 134 = Router Advertisement

# Request RA explicitly
rdisc6 eth0    # Shows received RA info
# or
ndisc6 2001:db8::1 eth0    # Probe specific IPv6 address

# If no RA: check SLAAC is enabled
cat /proc/sys/net/ipv6/conf/eth0/accept_ra
# 0 = disabled, 1 = accept (but not on forwarding devices), 2 = always
```

## Step 2: Check DHCPv6 (if using DHCPv6 mode)

```bash
# Check if DHCPv6 client is running
ps aux | grep -E "dhclient|dhcpcd|networkd|NetworkManager"

# Check systemd-networkd DHCPv6 status
networkctl status eth0 | grep -A5 "DHCPv6"

# Check NetworkManager IPv6 method
nmcli connection show eth0 | grep ipv6.method
# Should be: ipv6.method: auto (SLAAC/DHCPv6) or dhcp (DHCPv6-stateful)

# Manually trigger DHCPv6 request
dhclient -6 -v eth0 2>&1 | head -30
# Should show SOLICIT, ADVERTISE, REQUEST, REPLY messages

# Check DHCPv6 traffic
tcpdump -i eth0 -n "udp and (port 546 or port 547)"
# If no REPLY received: DHCPv6 server unreachable or not configured
```

## Step 3: NDP and Gateway Reachability

```bash
# Check NDP neighbor table
ip -6 neigh show
# REACHABLE = working
# STALE = was reachable, needs refresh
# FAILED = NDP has failed

# If gateway is FAILED:
ip -6 neigh show | grep "2001:db8::1"
# If absent or FAILED: ping the gateway
ping6 2001:db8::1

# If ping fails: check if gateway MAC is resolvable
# Manually send NDP probe
arping6 -I eth0 -c3 2001:db8::1    # If arping6 is available
# or
ndisc6 2001:db8::1 eth0

# If NDP fails at the VM but works on hypervisor host:
# → Bridge netfilter is blocking NDP
# On KVM host:
sysctl net.bridge.bridge-nf-call-ip6tables
# If = 1: check ip6tables FORWARD rules
ip6tables -L FORWARD -n -v | grep DROP
```

## Step 4: IPv6 Routing

```bash
# Check default IPv6 route
ip -6 route show | grep default
# Expected: default via 2001:db8::1 dev eth0 proto ra metric 100

# If no default route: RA not received or SLAAC disabled
# Try forcing RA acceptance (even with forwarding enabled)
sysctl -w net.ipv6.conf.eth0.accept_ra=2

# If route is present but external doesn't work:
# Check the hypervisor host has IPv6 forwarding enabled (for NAT/routing VMs)
ssh hypervisor-host "sysctl net.ipv6.conf.all.forwarding"

# Trace route to external IPv6
traceroute6 2001:4860:4860::8888
# Each hop should respond; * * * indicates firewall or routing issue
```

## Step 5: MTU and PMTUD

```bash
# Test MTU with different packet sizes
ping6 -s 100 2001:4860:4860::8888    # Small - should work
ping6 -s 1400 2001:4860:4860::8888  # Large - may fail if MTU issue
ping6 -M do -s 1452 2001:4860:4860::8888  # DF-bit set

# If large packets fail: MTU mismatch (likely due to overlay encapsulation)
ip link show eth0 | grep mtu
# If overlay (VXLAN, Geneve): effective MTU = physical - 50-100 bytes

# Fix: lower MTU in VM
ip link set eth0 mtu 1450

# Or configure rp_filter and ICMPv6 Packet Too Big
tcpdump -i eth0 -n "icmp6 and icmp6[0] == 2"    # Watch for PTB messages
```

## Hypervisor-Specific Quick Fixes

```bash
# KVM: restart libvirt network (resets radvd and dnsmasq)
virsh net-destroy default
virsh net-start default

# Proxmox: restart networking to re-apply bridge config
systemctl restart networking

# Check bridge MDB for multicast snooping issues
bridge mdb show dev br0 | grep "ff02"
# Should show ff02::1 (all-nodes) and ff02::2 (all-routers)
# If missing: multicast snooping may be dropping RA multicast

# Disable multicast snooping as a test
echo 0 > /sys/devices/virtual/net/br0/bridge/multicast_snooping
# If IPv6 starts working: multicast snooping misconfiguration is the cause
```

## Diagnostic Script

```bash
#!/bin/bash
# /usr/local/bin/diagnose-ipv6-vm.sh
# Run inside the VM to collect IPv6 diagnostic information

echo "=== IPv6 Addresses ==="
ip -6 addr show

echo "=== IPv6 Routes ==="
ip -6 route show

echo "=== NDP Neighbors ==="
ip -6 neigh show

echo "=== Router Advertisements (5 second capture) ==="
timeout 5 tcpdump -i eth0 -n "icmp6 and icmp6[0] == 134" 2>/dev/null || echo "No RA received"

echo "=== DHCPv6 Status ==="
systemctl status NetworkManager | grep -i "ipv6\|dhcpv6" | tail -5

echo "=== sysctl IPv6 settings ==="
sysctl net.ipv6.conf.eth0.accept_ra
sysctl net.ipv6.conf.all.forwarding

echo "=== Ping gateway ==="
GATEWAY=$(ip -6 route show default | awk '{print $3}')
ping6 -c2 "$GATEWAY" 2>&1 | tail -3
```

## Conclusion

Troubleshooting IPv6 in VM environments follows a top-down approach: verify address assignment (SLAAC from RA or DHCPv6), confirm NDP resolves the gateway, check the default route is present, and test external connectivity with progressively larger packets to detect MTU issues. The most common causes are: Linux bridge netfilter blocking NDP (fix: `bridge-nf-call-ip6tables=0`), bridge multicast snooping dropping RA multicast (fix: disable snooping or add MDB entries), missing IPv6 forwarding on the host (fix: `net.ipv6.conf.all.forwarding=1`), and MTU mismatches in overlay networks (fix: reduce VM MTU to 1450). Capture NDP traffic at both the VM interface and the hypervisor bridge to identify where packets are being dropped.
