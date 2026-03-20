# How to Create a Bridge with ip link add type bridge

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, ip command, iproute2, Bridge, Networking

Description: Create a Linux network bridge using ip link add type bridge, add member interfaces, assign an IP address, and configure bridge properties.

## Introduction

`ip link add type bridge` creates a software bridge (virtual switch) that connects multiple network interfaces at Layer 2. Frames are forwarded based on MAC addresses. This is the low-level approach using iproute2 directly, without NetworkManager or Netplan.

## Create a Bridge Interface

```bash
# Create bridge br0
ip link add name br0 type bridge

# Bring it up
ip link set br0 up
```

## Add Physical Interfaces to the Bridge

```bash
# Attach eth0 to bridge br0
# First remove any IP from eth0
ip addr flush dev eth0
ip link set eth0 master br0
ip link set eth0 up

# Attach eth1 to the same bridge
ip addr flush dev eth1
ip link set eth1 master br0
ip link set eth1 up
```

## Assign an IP to the Bridge

```bash
# The bridge gets the IP, not the member interfaces
ip addr add 192.168.1.10/24 dev br0
ip route add default via 192.168.1.1
```

## Verify the Bridge

```bash
# Show bridge interfaces
ip link show type bridge

# Show bridge members
bridge link show

# Show IP on bridge
ip addr show br0

# Show MAC forwarding table
bridge fdb show dev br0
```

## Enable STP on the Bridge

```bash
# Enable Spanning Tree Protocol
ip link set br0 type bridge stp_state 1

# Disable STP
ip link set br0 type bridge stp_state 0
```

## Set Bridge Properties

```bash
# Set forward delay (default 15 seconds)
ip link set br0 type bridge forward_delay 400

# Set hello time (centiseconds)
ip link set br0 type bridge hello_time 200

# Set max age
ip link set br0 type bridge max_age 2000
```

## Remove Interface from Bridge

```bash
# Detach eth1 from bridge
ip link set eth1 nomaster
```

## Delete the Bridge

```bash
# First remove all members
ip link set eth0 nomaster
ip link set eth1 nomaster

# Delete the bridge
ip link set br0 down
ip link delete br0
```

## Conclusion

`ip link add name br0 type bridge` creates a bridge, and `ip link set <interface> master br0` attaches member interfaces. The bridge device gets the IP address — not the members. Enable STP with `ip link set br0 type bridge stp_state 1`. Use `bridge link show` to verify members and `bridge fdb show` for the MAC table.
