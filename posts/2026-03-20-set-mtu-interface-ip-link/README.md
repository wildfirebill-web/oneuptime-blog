# How to Set the MTU on an Interface with ip link

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, ip command, iproute2, MTU, Networking, Performance

Description: Set the MTU (Maximum Transmission Unit) on a Linux network interface using ip link set mtu, including jumbo frames, tunnel MTU, and persistent MTU configuration.

## Introduction

MTU (Maximum Transmission Unit) is the largest packet size the interface can transmit. The default is 1500 bytes for Ethernet. Jumbo frames use 9000 bytes for better throughput on storage networks. Tunnels (GRE, VXLAN) require smaller MTUs to account for encapsulation overhead.

## Show Current MTU

```bash
# Show MTU for all interfaces

ip link show

# Show MTU for a specific interface
ip link show eth0

# Sample output (MTU appears as "mtu 1500"):
# 2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
```

## Set MTU

```bash
# Set MTU to 9000 (jumbo frames)
ip link set eth0 mtu 9000

# Set standard MTU
ip link set eth0 mtu 1500

# Verify
ip link show eth0 | grep mtu
```

## Set MTU for Tunnel Interfaces

```bash
# GRE overhead is 24 bytes, VXLAN is 50 bytes
# Set MTU 24 bytes less than underlay for GRE
ip link set gre1 mtu 1476

# Set MTU 50 bytes less than underlay for VXLAN
ip link set vxlan0 mtu 1450
```

## Calculate Correct Tunnel MTU

```text
Ethernet underlay MTU: 1500
GRE overhead:           24 bytes → tunnel MTU = 1476
VXLAN overhead:         50 bytes → tunnel MTU = 1450
WireGuard overhead:     60 bytes → tunnel MTU = 1440
IPsec (ESP) overhead:  ~73 bytes → tunnel MTU = 1427
```

## Set MTU on a VLAN Interface

```bash
# VLAN MTU should match or be less than parent interface MTU
ip link set eth0.10 mtu 1500
```

## Set MTU at Interface Creation (Permanent)

```bash
# systemd-networkd (persistent)
# /etc/systemd/network/10-eth0.network
# [Link]
# MTUBytes=9000

# Netplan (persistent)
# ethernets:
#   eth0:
#     mtu: 9000

# nmcli (persistent)
nmcli connection modify "eth0" ethernet.mtu 9000
```

## Verify MTU End-to-End

```bash
# Test if large packets get through (no fragmentation)
# Set DF bit and send 1472 byte payload (= 1500 MTU - 28 IP/ICMP headers)
ping -M do -s 1472 192.168.1.1

# If fragmentation needed, reduce until it works
ping -M do -s 1400 192.168.1.1
```

## Conclusion

`ip link set <interface> mtu <size>` changes the MTU immediately. Standard Ethernet uses 1500; jumbo frames use 9000. Tunnel interfaces need MTU reductions to account for encapsulation headers. For persistence, configure MTU in Netplan, nmcli, or systemd-networkd.
