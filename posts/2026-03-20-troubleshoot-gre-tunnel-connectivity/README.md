# How to Troubleshoot GRE Tunnel Connectivity Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, GRE, Tunnel, Troubleshooting, Networking, tcpdump, iproute2

Description: Diagnose and fix GRE tunnel connectivity problems including missing kernel modules, firewall blocking, routing issues, and MTU fragmentation.

## Introduction

GRE tunnel issues typically fall into four categories: the tunnel interface is not up, the underlay cannot reach the remote host, a firewall is blocking GRE protocol packets, or routing is incorrect. This guide walks through a systematic debugging process.

## Step 1: Check Tunnel Interface State

```bash
# Check if the tunnel interface is UP
ip link show gre0
# Look for: state UP LOWER_UP

# Check IP assignment
ip addr show gre0

# If DOWN, bring it up
ip link set gre0 up
```

## Step 2: Test Underlay Connectivity

The GRE tunnel requires IP connectivity between the local and remote underlay IPs:

```bash
# Test connectivity to the remote underlay IP
ping -c 3 10.0.0.2

# If this fails, the GRE tunnel cannot work
# Fix the underlay routing first
ip route get 10.0.0.2
```

## Step 3: Check for Firewall Blocking GRE

GRE uses IP protocol 47. Firewalls often block it:

```bash
# Check if iptables is blocking GRE
iptables -L -n -v | grep -i gre

# Allow GRE protocol on both hosts
iptables -A INPUT -p gre -j ACCEPT
iptables -A OUTPUT -p gre -j ACCEPT
iptables -A FORWARD -p gre -j ACCEPT

# For nftables
nft add rule inet filter input ip protocol 47 accept
nft add rule inet filter output ip protocol 47 accept
```

## Step 4: Check ip_gre Module

```bash
# Verify the module is loaded
lsmod | grep ip_gre

# If not loaded, load it
modprobe ip_gre
```

## Step 5: Capture GRE Traffic

```bash
# On the local host, capture GRE traffic on the physical interface
# GRE uses IP protocol 47
tcpdump -i eth0 proto gre -n

# Check if packets are being sent (TX) and received (RX)
# If TX but no RX from remote — firewall on remote or routing issue
```

## Step 6: Verify Tunnel Configuration

```bash
# Show detailed tunnel configuration
ip -d tunnel show gre0

# Verify local and remote IPs match what is expected
# local=10.0.0.1 remote=10.0.0.2

# Check IP forwarding
sysctl net.ipv4.ip_forward
```

## Step 7: Check Routing Through the Tunnel

```bash
# Test if routing decision uses the tunnel
ip route get 192.168.2.1
# Should show: via 172.16.0.2 dev gre0

# If route is missing, add it
ip route add 192.168.2.0/24 via 172.16.0.2

# Test end-to-end ping
ping -c 3 192.168.2.1
```

## Common GRE Issues Summary

| Symptom | Cause | Fix |
|---|---|---|
| Tunnel interface DOWN | Underlay unreachable | Fix underlay routing |
| Tunnel up, no traffic | GRE blocked by firewall | Allow protocol 47 |
| Ping works, TCP fails | MTU too large | Set MTU 1476, add MSS clamping |
| Traffic lost in one direction | Asymmetric routing | Check routes on both hosts |
| ip_gre not found | Module not loaded | `modprobe ip_gre` |

## Check Tunnel Statistics

```bash
# Show TX/RX packets and bytes
ip -s link show gre0

# If TX increases but RX stays at 0, remote end is not sending back
# If both stay at 0, packets are not reaching the tunnel
```

## Conclusion

GRE tunnel troubleshooting follows: check interface state, verify underlay IP connectivity, ensure firewall allows protocol 47, verify the `ip_gre` module is loaded, and check routing. Use `tcpdump proto gre` on the physical interface to see if GRE packets are being exchanged. MTU issues (large TCP failing while small pings work) indicate you need MSS clamping or reduced MTU.
