# How to Identify MTU Mismatch Issues on Network Interfaces

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MTU, Mismatch, Networking, Linux, Troubleshooting, Interface

Description: Identify MTU mismatches between network interfaces, tunnel endpoints, and network paths that cause fragmentation, packet drops, and performance degradation.

## Introduction

MTU mismatches occur when different parts of a network path expect different maximum packet sizes. Common scenarios: a host configured for 9000 MTU (jumbo frames) connecting to a switch that only supports 1500 MTU, or a VPN endpoint with wrong MTU that silently drops large packets. Identifying mismatches requires checking MTU at each hop and comparing with actual test packets.

## Check Local Interface MTU

```bash
# List all interfaces with their MTUs:
ip link show | grep -E "^[0-9]|mtu"
# Shows: mtu 1500 for each interface

# Or more readable:
ip link show | awk '/^[0-9]/{iface=$2} /mtu/{match($0, /mtu ([0-9]+)/, m); print iface, "MTU:", m[1]}'

# Check specific interface:
ip link show eth0 | grep mtu
cat /sys/class/net/eth0/mtu

# Check all tunnel interfaces:
ip link show | grep -E "tun|gre|vxlan|wireguard|wg"
```

## Detect MTU Mismatch with ping

```bash
# If two hosts have different MTUs, one can't send max-size packets to the other:

# Test from HOST A to HOST B:
# Host A has MTU 9000 (jumbo frames):
ping -M do -s 8972 -c 3 HOST_B   # 9000 - 28 = 8972
# If HOST B or the path between them doesn't support jumbo: fail

# Test from HOST B to HOST A:
# Host B has MTU 1500:
ping -M do -s 1472 -c 3 HOST_A   # 1500 - 28 = 1472
# Should succeed (both support 1500)

# Asymmetric success → MTU mismatch:
# A → B works at 1472 but not 8972: path MTU is 1500
# B → A fails at 1472: interface or path MTU < 1500
```

## Find MTU Mismatch in a Path

```bash
# Use tracepath to find MTU at each hop:
tracepath -n 10.20.0.5
# Shows "pmtu" changes at each hop

# Example output showing mismatch:
# 1: 10.0.0.1      0.5ms
# 2: 192.168.1.1   2.1ms  pmtu 9000   ← Core switch (jumbo frames)
# 3: 10.1.0.1      3.5ms  pmtu 1500   ← This hop reduces MTU!
# 4: 10.20.0.5     5.1ms  reached

# The mismatch is between hop 2 (9000) and hop 3 (1500)
# If hop 3 doesn't support jumbo frames and hop 2 forwards jumbo packets:
# → Packets fragmented (if DF bit not set) or dropped (if DF bit set)
```

## Common MTU Mismatch Scenarios

```bash
# Scenario 1: Jumbo frame NIC + non-jumbo switch port
# Symptom: large transfers fail, small transfers work
# Check: NIC MTU
ip link show eth0 | grep mtu
# vs switch port MTU (check switch configuration)

# Scenario 2: Docker container MTU mismatch
# Docker default bridge MTU can differ from host MTU
docker network inspect bridge | grep Mtu
# vs:
ip link show eth0 | grep mtu
# If bridge MTU > host MTU: container packets get dropped

# Scenario 3: VPN tunnel MTU not updated when path changes
ip link show wg0 | grep mtu    # Should be < underlying interface MTU
ip link show eth0 | grep mtu   # Underlying interface
# WireGuard MTU should be eth0_MTU - 80

# Scenario 4: VLAN interface inheriting wrong MTU
ip link show eth0.100 | grep mtu  # VLAN interface
ip link show eth0 | grep mtu      # Parent interface
# Both should have same MTU (VLAN adds 4 bytes tag but MTU remains same)
```

## Verify Two Hosts Can Exchange Max-Size Packets

```bash
#!/bin/bash
# Verify MTU match between two hosts

HOST1="10.20.0.10"
HOST2="10.20.0.11"

echo "Testing MTU between $HOST1 and $HOST2"
echo "========================================"

# Test standard Ethernet MTU (1500):
echo -n "1500 MTU: "
if ping -M do -s 1472 -c 3 -W 2 $HOST2 > /dev/null 2>&1; then
    echo "PASS"
else
    echo "FAIL"
fi

# Test jumbo frames (9000):
echo -n "9000 MTU: "
if ping -M do -s 8972 -c 3 -W 2 $HOST2 > /dev/null 2>&1; then
    echo "PASS (jumbo frames working)"
else
    echo "FAIL (jumbo frames not supported on this path)"
fi

# Find actual path MTU:
ACTUAL_MTU=$(tracepath -n $HOST2 | grep pmtu | tail -1 | grep -oP 'pmtu \K[0-9]+')
echo "Path MTU from tracepath: ${ACTUAL_MTU:-not determined}"
```

## Conclusion

MTU mismatches are identified by testing packet delivery at different sizes with the DF bit set. Use `tracepath` to find where the MTU changes. Common mismatches are jumbo-enabled hosts connecting to standard-MTU switches, Docker containers with wrong MTU settings, and VPN tunnels not accounting for tunnel overhead. Fix mismatches by aligning MTU values across all interfaces in the path, or by reducing the MTU of the sending interface to match the bottleneck.
