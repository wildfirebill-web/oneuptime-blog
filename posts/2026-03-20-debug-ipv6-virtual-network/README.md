# How to Debug IPv6 in Virtualized Network Environments

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Debugging, Virtualization, KVM, VMware, Network Troubleshooting

Description: Systematically debug IPv6 connectivity issues in virtualized environments, including NDP failures, Router Advertisement problems, MTU issues, and hypervisor-specific networking quirks.

## Introduction

Debugging IPv6 in virtualized environments requires understanding both the guest OS networking stack and the hypervisor's virtual switching layer. Common issues include hypervisors blocking NDP multicast, bridge netfilter rules dropping ICMPv6, incorrect MTU settings causing PMTUD failures, and VM NIC settings that prevent proper IPv6 operation.

## Diagnostic Checklist

```bash
# Start with these checks on both host and guest:

# 1. Is IPv6 enabled on the interface?

ip -6 addr show
# Should have at least a link-local (fe80::) address

# 2. Is there a default route?
ip -6 route show default

# 3. Can we reach the gateway?
ping6 fe80::1%eth0    # Link-local requires scope

# 4. Is NDP working?
ip -6 neigh show
# Should show gateway with MAC and REACHABLE state

# 5. Are Router Advertisements arriving?
tcpdump -i eth0 -n "icmp6 and icmp6[0] == 134"

# 6. Is MTU correct?
ip link show eth0 | grep mtu
```

## Debug NDP (Neighbor Discovery Protocol) Failures

```bash
# NDP failure: VM cannot resolve IPv6 neighbors

# Step 1: Check if NS/NA packets are being sent
# From VM:
tcpdump -i eth0 -n "icmp6 and (icmp6[0] == 135 or icmp6[0] == 136)"
# 135 = Neighbor Solicitation (sent by VM)
# 136 = Neighbor Advertisement (sent by target)
# If no NA received: NDP is blocked somewhere

# Step 2: Check the hypervisor host
# On KVM host: check bridge is forwarding ICMPv6
tcpdump -i br0 -n "icmp6 and (icmp6[0] == 135 or icmp6[0] == 136)"
# Should see same traffic as inside VM

# Step 3: Check bridge-nf on KVM host
sysctl net.bridge.bridge-nf-call-ip6tables
# If = 1 and ip6tables has DROP: NDP is blocked
ip6tables -L FORWARD -n -v | grep DROP
# Fix: set bridge-nf-call-ip6tables=0 or add ACCEPT rules for ICMPv6
```

## Debug Router Advertisement Issues

```bash
# RA failure: VM gets no global IPv6 address (SLAAC not working)

# From VM: listen for Router Advertisements
tcpdump -i eth0 -n -v "icmp6 and icmp6[0] == 134"
# Should see periodic RA from router (every 4-7 minutes, or on RS)

# If no RA received: send a Router Solicitation
# rdisc6 sends RS and shows received RA
rdisc6 eth0
# Expected: shows prefix, lifetime, flags from RA

# If RA is seen on host but not in VM:
# Check hypervisor is forwarding multicast to the VM

# For KVM/Linux bridge: check multicast snooping
bridge link show dev br0
# disable multicast snooping if it's blocking RA multicast
bridge mdb show dev br0
# ff02::1 (all-nodes) should be in the MDB or multicast snooping disabled

# Disable multicast snooping on bridge (last resort)
echo 0 > /sys/devices/virtual/net/br0/bridge/multicast_snooping
```

## Debug IPv6 Connectivity Through Virtual Switches

```bash
# Use tcpdump at multiple points to trace IPv6 flow

# 1. Inside VM (source)
tcpdump -i eth0 -n "ip6 and host 2001:db8::1"

# 2. On hypervisor bridge
tcpdump -i br0 -n "ip6 and host 2001:db8::1"

# 3. On physical interface
tcpdump -i eth0 -n "ip6 and host 2001:db8::1"

# If packet is seen in VM but not on bridge:
# → VM NIC is not connected to bridge (check virsh domiflist / VBoxManage)

# If packet is seen on bridge but not on eth0:
# → ip6tables FORWARD rules may be dropping it
ip6tables -L FORWARD -n -v
```

## Debug MTU Issues in Virtual Networks

```bash
# MTU issues cause intermittent failures (large packets fail, small succeed)

# Test with varying packet sizes
ping6 -s 100 2001:4860:4860::8888    # Should work
ping6 -s 1400 2001:4860:4860::8888  # May fail if MTU issue
ping6 -M do -s 1452 2001:4860:4860::8888  # With DF bit

# Check interface MTU in VM
ip link show eth0 | grep mtu
# Default: 1500, may be lower in VM (1450 for VXLAN encap)

# Check for ICMPv6 Packet Too Big messages
tcpdump -i eth0 -n "icmp6 and icmp6[0] == 2"
# If PTB messages are being received but ignored:
# → Check if ICMPv6 is blocked by ip6tables
ip6tables -L INPUT -n -v | grep icmpv6

# Fix: reduce VM interface MTU to account for encap overhead
ip link set eth0 mtu 1450   # For VXLAN (1500 - 50 overhead)
```

## VMware-Specific IPv6 Debugging

```bash
# ESXi: check if VMkernel is consuming IPv6 traffic
esxcli network ip interface ipv6 address list

# Check if vSwitch security policies are blocking IPv6
# via vSphere Client: vSwitch → Properties → Security:
# - Promiscuous Mode: Reject (should be OK)
# - MAC Address Changes: Accept (needed for some scenarios)
# - Forged Transmits: Accept (needed for nested virtualization)

# Check ESXi firewall for IPv6
esxcli network firewall ruleset list | grep ipv6

# VMkernel ping6 test
esxcli network diag ping --host 2001:db8::1 --netstack default --ipv6
```

## Hyper-V IPv6 Debugging

```powershell
# Check VM NIC settings that affect IPv6
Get-VMNetworkAdapter -VMName "MyVM" | Format-List *Guard*

# Router Guard blocks RA (disable if VM is an IPv6 router)
Get-VMNetworkAdapter -VMName "MyVM" | Select-Object RouterGuard

# DHCP Guard blocks DHCPv6 (disable if VM is a DHCPv6 server)
Get-VMNetworkAdapter -VMName "MyVM" | Select-Object DhcpGuard

# Test IPv6 from Hyper-V host to VM
Test-NetConnection -ComputerName "2001:db8::100" -TraceRoute

# Check Windows Filtering Platform (WFP) for IPv6 drops
netsh wfp show boottimepolicy
```

## Conclusion

Debugging IPv6 in virtualized environments requires a layer-by-layer approach: verify IPv6 addresses in the guest, trace NDP traffic through the hypervisor bridge, check bridge netfilter and ip6tables rules, and validate MTU consistency. The most common causes are Linux bridge netfilter blocking NDP when `bridge-nf-call-ip6tables=1`, Hyper-V Router Guard blocking Router Advertisements, VMware MAC Address Changes policies interfering with NDP, and MTU mismatches causing PMTUD to silently fail. Always use `tcpdump` at both the guest NIC and the hypervisor bridge to identify where traffic is being dropped.
