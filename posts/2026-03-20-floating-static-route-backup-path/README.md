# How to Configure a Floating Static Route as a Backup Path

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Networking, Routing, Static Routes, Failover, Linux, IPv4

Description: Configure a floating static route with a higher metric to serve as an automatic backup path when the primary route becomes unavailable.

## Introduction

A floating static route is a static route configured with a higher administrative distance (or metric) than the primary path. Under normal conditions, the primary route is preferred and the floating route is not used. If the primary route disappears from the routing table (due to link failure or dynamic protocol withdrawal), the floating route automatically activates.

## How Floating Routes Work

The routing table always installs the route with the lowest metric. By giving the backup route a higher metric, it stays dormant until the primary is gone:

```text
Primary:  10.20.0.0/24 via 192.168.1.1  metric 10   (active)
Backup:   10.20.0.0/24 via 192.168.2.1  metric 100  (floating, dormant)
```

When the primary link fails and the 10.20.0.0/24 via 192.168.1.1 route is removed, Linux automatically installs the backup.

## Configuring the Primary and Floating Routes

```bash
# Primary route - lower metric, preferred path

ip route add 10.20.0.0/24 via 192.168.1.1 metric 10

# Floating/backup route - higher metric, only used if primary is gone
ip route add 10.20.0.0/24 via 192.168.2.1 metric 100

# Verify both routes are installed
ip route show 10.20.0.0/24
# 10.20.0.0/24 via 192.168.1.1 dev eth0 metric 10
# 10.20.0.0/24 via 192.168.2.1 dev eth1 metric 100
```

## Simulating Failover

```bash
# Simulate primary link failure by bringing down eth0
ip link set eth0 down

# The primary route disappears, backup route becomes active
ip route show 10.20.0.0/24
# 10.20.0.0/24 via 192.168.2.1 dev eth1 metric 100

# Verify connectivity still works through backup
ping -c 4 10.20.0.5

# Restore the primary link
ip link set eth0 up
# Primary route returns and takes over automatically
```

## Making Floating Routes Persistent

With systemd-networkd:

```ini
# /etc/systemd/network/10-primary.network
[Match]
Name=eth0

[Network]
Address=192.168.1.2/24

[Route]
Destination=10.20.0.0/24
Gateway=192.168.1.1
Metric=10
```

```ini
# /etc/systemd/network/20-backup.network
[Match]
Name=eth1

[Network]
Address=192.168.2.2/24

[Route]
Destination=10.20.0.0/24
Gateway=192.168.2.1
Metric=100
```

## Combining with Dynamic Routing

When using OSPF, a floating static default route can serve as a backup to the OSPF-learned default:

```bash
# OSPF default route has AD 110 in Cisco, but on Linux FRR it has a metric
# Add a static default with metric higher than OSPF's effective metric
ip route add default via 203.0.113.1 metric 200

# FRR OSPF will inject a better metric default route when available
# If OSPF default disappears, the static at metric 200 takes over
```

## Monitoring Failover

```bash
# Watch the routing table for changes in real time
watch -n 2 "ip route show 10.20.0.0/24"

# Log route changes using iproute2 monitor
ip monitor route | grep "10.20.0.0/24"
```

## Conclusion

Floating static routes are a simple, zero-protocol-overhead method for route failover. They are ideal for dual-ISP setups, WAN backup links, and any scenario where you want a deterministic failover without running a full dynamic routing protocol. For more intelligent failover that can detect remote failures (not just local link failures), combine with BFD or a monitoring daemon.
