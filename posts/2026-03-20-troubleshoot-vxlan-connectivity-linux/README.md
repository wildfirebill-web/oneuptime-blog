# How to Troubleshoot VXLAN Connectivity on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, VXLAN, Troubleshooting, Networking, tcpdump, FDB, Overlay Network

Description: Systematically diagnose VXLAN overlay network connectivity issues including firewall blocking, missing FDB entries, MTU problems, and VTEP configuration errors.

## Introduction

VXLAN connectivity issues usually stem from: UDP port 4789 being blocked, incorrect VTEP IPs, missing FDB flood entries, MTU fragmentation, or the underlay network not supporting the required connectivity. This guide provides a step-by-step troubleshooting process.

## Step 1: Verify Underlay Connectivity

VXLAN requires UDP/4789 to work between VTEPs:

```bash
# Test underlay reachability
ping -c 3 10.0.0.2

# Test UDP port 4789 is reachable (from another host)
nc -u 10.0.0.2 4789
# Or
nmap -sU -p 4789 10.0.0.2
```

## Step 2: Check Firewall

```bash
# VXLAN uses UDP port 4789
iptables -L -n -v | grep 4789

# If no rule allows it, add one
iptables -A INPUT -p udp --dport 4789 -j ACCEPT
```

## Step 3: Check VXLAN Interface State

```bash
# Verify VXLAN interface is UP
ip link show vxlan0
# Should show: state UP LOWER_UP

# Check the VNI and remote VTEP
ip -d link show vxlan0
# Should show matching VNI and VTEP IPs
```

## Step 4: Check FDB Entries

```bash
# Verify flood entries exist for remote VTEPs
bridge fdb show dev vxlan0 | grep "00:00:00:00:00:00"

# If missing, add them
bridge fdb append 00:00:00:00:00:00 dev vxlan0 dst 10.0.0.2 permanent
```

## Step 5: Capture VXLAN Traffic

```bash
# On the source host, verify VXLAN packets are being sent
tcpdump -i eth0 udp port 4789 -n

# If you see packets sent but none received from remote:
# - Firewall on remote host is blocking
# - Remote VTEP IP is wrong

# On the remote host:
tcpdump -i eth0 udp port 4789 -n
# If packets arrive but overlay doesn't work, check VXLAN configuration on remote
```

## Step 6: Verify VNI Matches

Both ends must use the same VNI:

```bash
# Check VNI on local VXLAN interface
ip -d link show vxlan0 | grep "vxlan id"
# vxlan id 100 dev eth0 ...

# Must match on the remote host
# A VNI mismatch causes silent drops
```

## Step 7: Check MTU

```bash
# Check current VXLAN MTU
ip link show vxlan0 | grep mtu
# Should be 1450

# Test with large packet
ping -s 1422 -M do -c 3 10.200.0.2

# If this fails but small pings work → MTU issue
ip link set vxlan0 mtu 1450
```

## Step 8: Verify Bridge Configuration

```bash
# Check VXLAN is attached to bridge
bridge link show
# Should show: vxlan0: ... master br-overlay

# Check bridge is UP
ip link show br-overlay
```

## Common Issues Table

| Symptom | Cause | Fix |
|---|---|---|
| No VXLAN packets sent | Interface DOWN | `ip link set vxlan0 up` |
| Packets sent, none received | Firewall blocking | Allow UDP 4789 |
| Pings work, TCP fails | MTU too large | Set MTU 1450, MSS clamp |
| Asymmetric traffic | Missing FDB entry | Add remote VTEP to FDB |
| Wrong traffic segment | VNI mismatch | Match VNI on both sides |

## Conclusion

VXLAN troubleshooting starts with the underlay (can hosts communicate via UDP 4789?), then checks the overlay configuration (VNI, FDB entries, interface state). tcpdump on the physical interface filtering `udp port 4789` is the key diagnostic tool. Always verify that both sides have matching VNIs and correct VTEP IPs in their FDB flood entries.
