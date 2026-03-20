# How to Configure Balance-XOR Bonding (Mode 2) on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Network Bonding, Balance-XOR, Mode 2, Load Balancing, Networking

Description: Configure Linux balance-XOR bonding (mode 2) to distribute traffic across slave interfaces using a hashing algorithm for consistent per-flow load balancing.

## Introduction

Balance-XOR (mode 2) uses a hash-based algorithm to select which slave interface transmits each packet. The same XOR hash ensures that packets belonging to the same flow always use the same slave, preventing out-of-order delivery. This mode provides both load balancing and fault tolerance without requiring switch-side LACP.

## How Balance-XOR Works

The transmit slave is selected using:
```text
slave = hash(src_MAC XOR dst_MAC) % num_slaves
```

With `layer3+4` policy:
```text
slave = hash(src_IP, dst_IP, src_port, dst_port) % num_slaves
```

## Configure Balance-XOR

```bash
# Load the bonding module

modprobe bonding

# Create bond in balance-xor mode
ip link add bond0 type bond mode balance-xor

# Set hash policy (layer3+4 is recommended for IP traffic)
ip link set bond0 type bond xmit_hash_policy layer3+4

# Set MII monitoring
ip link set bond0 type bond miimon 100

# Add slave interfaces
ip link set eth0 down
ip link set eth1 down
ip link set eth0 master bond0
ip link set eth1 master bond0

# Bring up the bond
ip link set bond0 up
ip addr add 192.168.1.100/24 dev bond0
ip route add default via 192.168.1.1
```

## Verify Balance-XOR Mode

```bash
cat /proc/net/bonding/bond0
# Bonding Mode: load balancing (xor)
# Transmit Hash Policy: layer3+4 (3)
```

## Hash Policy Options

```bash
# Layer2 (default): based on src/dst MAC addresses
ip link set bond0 type bond xmit_hash_policy layer2

# Layer2+3: based on MAC + IP
ip link set bond0 type bond xmit_hash_policy layer2+3

# Layer3+4: based on IP + port - best for most TCP/UDP workloads
ip link set bond0 type bond xmit_hash_policy layer3+4
```

## Persistent Configuration

```yaml
# Netplan configuration
bonds:
  bond0:
    interfaces: [eth0, eth1]
    addresses: [192.168.1.100/24]
    parameters:
      mode: balance-xor
      mii-monitor-interval: 100
      transmit-hash-policy: layer3+4
```

## Balance-XOR vs Round-Robin vs LACP

| Feature | Mode 0 (RR) | Mode 2 (XOR) | Mode 4 (LACP) |
|---|---|---|---|
| Load distribution | Per-packet (round-robin) | Per-flow (hash) | Per-flow (hash) |
| Out-of-order packets | Yes | No | No |
| Switch requirement | Static aggregation | Static aggregation | LACP |
| Fault tolerance | Yes | Yes | Yes |

## Switch Configuration Note

Balance-XOR requires switch-side static link aggregation (port-channel) configured on the ports connecting to the bond slaves. Without this, the switch may see MAC address flapping from different ports.

## Conclusion

Balance-XOR bonding provides consistent per-flow distribution using a configurable hash policy. It avoids out-of-order packets (unlike round-robin) while still distributing load across multiple interfaces. Use `layer3+4` hash policy for the best distribution of TCP/UDP traffic. Requires switch-side static aggregation but not LACP.
