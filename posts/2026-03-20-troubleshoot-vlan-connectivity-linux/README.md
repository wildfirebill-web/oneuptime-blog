# How to Troubleshoot VLAN Connectivity Issues on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, VLAN, Troubleshooting, Networking, tcpdump, iproute2, 802.1Q

Description: Diagnose and fix common VLAN connectivity problems on Linux including missing kernel modules, interface misconfiguration, switch trunk mismatches, and MTU issues.

## Introduction

VLAN connectivity issues often stem from a few common causes: the `8021q` module not loaded, VLAN interface not brought up, mismatched VLAN IDs between Linux and the switch, switch port not in trunk mode, or MTU problems. This guide provides a systematic troubleshooting approach.

## Step 1: Verify the 8021q Module is Loaded

```bash
# Check if 8021q is loaded
lsmod | grep 8021q

# If not loaded, load it
modprobe 8021q

# Verify loading succeeded
lsmod | grep 8021q
```

## Step 2: Check Interface State

```bash
# Check if parent interface is UP
ip link show eth0
# Should show: state UP

# Check if VLAN interface is UP
ip link show eth0.100
# Should show: state UP

# If either is DOWN, bring it up
ip link set eth0 up
ip link set eth0.100 up
```

## Step 3: Verify IP Address Assignment

```bash
# Check IP address on the VLAN interface
ip addr show eth0.100

# If no IP, assign one
ip addr add 192.168.100.10/24 dev eth0.100
```

## Step 4: Check the Routing Table

```bash
# Verify a route exists for the VLAN subnet
ip route show | grep 192.168.100

# If missing, add the route
ip route add 192.168.100.0/24 dev eth0.100

# Check default gateway
ip route show default
```

## Step 5: Verify VLAN Tagging with tcpdump

```bash
# Generate a ping from the VLAN interface
ping -I eth0.100 -c 3 192.168.100.1 &

# Capture on parent interface — should see 802.1Q tags
tcpdump -i eth0 -e vlan 100 -n

# If you see frames but no response — the switch may not be forwarding VLAN 100
# If you see NO frames at all — the VLAN interface is not sending traffic
```

## Step 6: Check Switch Configuration

Common switch-side issues:
- Switch port not configured as trunk
- VLAN not allowed on the trunk port
- VLAN not created in the switch VLAN database

```bash
# Verify by checking what VLAN tags appear on the wire
tcpdump -i eth0 -e vlan -n -c 20

# If you see incoming tagged frames, the switch is sending VLAN traffic
# If you see no tagged frames, the switch port may be in access mode
```

## Step 7: Check ARP Resolution

```bash
# Check if ARP is resolving the gateway on the VLAN
ip neigh show dev eth0.100

# Force an ARP request
arping -I eth0.100 192.168.100.1
```

## Step 8: MTU Issues

802.1Q adds 4 bytes to the frame. If the MTU is exactly 1500, tagged frames may be dropped:

```bash
# Check current MTU
ip link show eth0.100

# Reduce MTU to 1496 if 802.1Q overhead causes issues
ip link set eth0.100 mtu 1496
```

## Common Issues Summary

| Symptom | Likely Cause | Fix |
|---|---|---|
| No VLAN interface | Module not loaded | `modprobe 8021q` |
| Interface DOWN | Not brought up | `ip link set eth0.100 up` |
| No ARP replies | Wrong VLAN ID | Verify VLAN ID vs switch |
| Frames visible, no reply | Switch trunk config | Check switch trunk config |
| Large frame drops | MTU mismatch | Reduce MTU to 1496 |

## Conclusion

Troubleshooting VLAN connectivity follows a top-down checklist: module loaded, interfaces up, IP assigned, route present, ARP working, and switch configured correctly. tcpdump on the parent interface is the most powerful verification tool — seeing `802.1Q vlan#<ID>` confirms that tagging is working on the Linux side.
