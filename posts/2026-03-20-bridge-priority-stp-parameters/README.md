# How to Configure Bridge Priority and STP Parameters

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Bridge, STP, Spanning Tree Protocol, Linux, brctl, ip link, Bridge Priority, RSTP

Description: Learn how to configure Spanning Tree Protocol (STP) parameters on Linux bridges including bridge priority, port cost, and port priority to control root bridge election and forwarding topology.

---

STP prevents Layer 2 loops in networks with multiple bridge paths. Configuring bridge priority determines which bridge becomes the root bridge and controls the active forwarding topology.

## STP Basics

```
Root Bridge: Lowest bridge ID (priority + MAC)
Bridge ID: Priority (0-65535, lower=better) + bridge MAC

Default priority: 32768
To become root bridge: set priority < 32768 (e.g., 4096)
```

## Setting Bridge Priority with brctl

```bash
# Set bridge priority (must be multiple of 4096)
brctl setbridgeprio br0 4096

# Verify
brctl showstp br0 | grep "bridge id"
# bridge id              1000.aabbccddeeff
# (1000 hex = 4096 decimal)
```

## Setting Bridge Priority with ip link

```bash
# Set bridge priority using ip link
ip link set br0 type bridge priority 4096

# Verify all bridge parameters
ip -d link show br0 | grep bridge
```

## Setting Port Cost

```bash
# Port cost influences path selection (lower cost = preferred path)
# Default: 100 for 100Mbps, 19 for 1Gbps, 4 for 10Gbps

# Set port cost on a bridge port
brctl setportcost br0 eth0 10    # Lower cost = preferred

# Or with ip link
ip link set eth0 type bridge_slave cost 10
```

## Setting Port Priority

```bash
# Port priority (0-255, lower = more preferred)
# Default: 128

brctl setportprio br0 eth0 64   # More preferred port

# With ip link
ip link set eth0 type bridge_slave priority 64
```

## Enabling RSTP (Rapid Spanning Tree)

```bash
# Enable RSTP (802.1w) instead of classic STP
ip link set br0 type bridge stp_state 1
# Note: Linux bridge always uses RSTP when STP is enabled via ip link

# Set hello time, max age, forward delay
ip link set br0 type bridge hello_time 200     # 2 seconds (200 * 10ms)
ip link set br0 type bridge max_age 2000       # 20 seconds
ip link set br0 type bridge forward_delay 1500 # 15 seconds
```

## Persistent STP Configuration (systemd-networkd)

```ini
# /etc/systemd/network/br0.netdev
[NetDev]
Name=br0
Kind=bridge

[Bridge]
STP=yes
Priority=4096
HelloTimeSec=2
MaxAgeSec=20
ForwardDelaySec=15
```

## Verifying STP State

```bash
# Show STP state for all ports
brctl showstp br0

# Output:
# br0
#  bridge id              1000.aabbccddeeff
#  designated root        1000.aabbccddeeff   ← This bridge IS the root
#  root port                0                 ← 0 = this is root bridge
#  path cost                  0
# 
# eth0 (1)
#  port id                8001
#  state                  forwarding          ← Active port
#  designated cost           0
#  port cost                 4

# Check per-port state
bridge link show
```

## Key Takeaways

- Lower bridge priority wins root bridge election; use priority 4096 for the designated root bridge.
- Bridge priority must be a multiple of 4096 (0, 4096, 8192, ..., 61440).
- Port cost controls path selection within STP; lower cost paths are preferred.
- Linux bridges use RSTP by default when STP is enabled, providing faster convergence than classic STP.
