# How to Delete an ARP Entry with ip neigh del

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ARP, Linux, Networking, ip-command, Troubleshooting, IPv4

Description: Remove stale or incorrect ARP cache entries on Linux using the ip neigh del command to resolve layer-2 connectivity issues.

## Introduction

The ARP (Address Resolution Protocol) cache maps IPv4 addresses to MAC addresses. Stale or incorrect entries can cause connectivity failures when a device changes its IP or MAC address. The `ip neigh del` command lets you surgically remove specific entries without flushing the entire cache.

## Viewing the ARP Cache

Before deleting, view the current ARP table:

```bash
# Show all ARP (neighbor) entries
ip neigh show

# Filter for a specific interface
ip neigh show dev eth0
```

Example output:
```
192.168.1.1 dev eth0 lladdr aa:bb:cc:dd:ee:ff REACHABLE
192.168.1.50 dev eth0 lladdr 11:22:33:44:55:66 STALE
```

## Deleting a Specific ARP Entry

Use `ip neigh del` with the IP address and interface:

```bash
# Delete the ARP entry for 192.168.1.50 on eth0
sudo ip neigh del 192.168.1.50 dev eth0
```

Verify the entry is gone:

```bash
ip neigh show | grep 192.168.1.50
```

## Flushing All Entries for an Interface

To remove all ARP entries on a specific interface:

```bash
# Flush all neighbor entries for eth0
sudo ip neigh flush dev eth0
```

To flush all STALE entries across all interfaces:

```bash
# Remove only entries in STALE state
sudo ip neigh flush nud stale
```

## ARP Entry States

| State | Meaning |
|-------|---------|
| REACHABLE | Recently confirmed active |
| STALE | Not confirmed recently but still usable |
| DELAY | Waiting to confirm reachability |
| FAILED | Resolution failed |
| PERMANENT | Statically configured, never expires |

## Adding a Static ARP Entry

To prevent a device from being wrongly resolved (e.g., after an IP migration):

```bash
# Add a permanent ARP entry
sudo ip neigh add 192.168.1.50 lladdr 11:22:33:44:55:66 dev eth0 nud permanent
```

## Troubleshooting Scenario

If a server was moved to a new NIC but the old MAC is cached:

```bash
# Delete stale entry
sudo ip neigh del 10.0.0.100 dev eth0

# Trigger re-resolution by pinging the host
ping -c 1 10.0.0.100

# Confirm new MAC is learned
ip neigh show | grep 10.0.0.100
```

## Conclusion

`ip neigh del` is an essential tool for resolving ARP-related connectivity problems. It gives precise control over the kernel's ARP cache without disrupting other entries on the interface.
