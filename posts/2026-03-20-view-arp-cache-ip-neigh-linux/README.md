# How to View ARP Cache Entries Using ip neigh on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Networking, ARP, ip neigh, IPv4, Network Diagnostics

Description: View, filter, add, and delete ARP cache entries on Linux using the ip neigh command, and understand the NUD (Neighbor Unreachability Detection) state machine.

## Introduction

The ARP cache (neighbor table) stores IP-to-MAC address mappings learned from ARP exchanges. `ip neigh` is the modern command to inspect and manage this table, replacing the legacy `arp` command.

## Viewing the ARP Cache

```bash
# Show all neighbor (ARP) entries
ip neigh show

# Show only IPv4 entries
ip -4 neigh show

# Show entries for a specific interface
ip neigh show dev eth0

# Show a specific host's entry
ip neigh show 192.168.1.1
```

Sample output:

```
192.168.1.1 dev eth0 lladdr aa:bb:cc:dd:ee:ff REACHABLE
192.168.1.50 dev eth0 lladdr 11:22:33:44:55:66 STALE
192.168.1.200 dev eth0  FAILED
```

## NUD State Machine

Every ARP entry cycles through states:

| State | Meaning |
|---|---|
| REACHABLE | Recently confirmed reachable |
| STALE | Not recently used, will be re-verified on next use |
| DELAY | Waiting for confirmation after use |
| PROBE | Sending ARP unicast probes to confirm reachability |
| FAILED | All probes failed — host unreachable |
| PERMANENT | Manually set, never ages out |
| NOARP | Interface does not use ARP (e.g., loopback) |

## Adding a Static ARP Entry

```bash
# Add a permanent ARP entry (does not age out)
sudo ip neigh add 192.168.1.50 lladdr aa:bb:cc:dd:ee:ff dev eth0 nud permanent

# Verify
ip neigh show 192.168.1.50
```

## Changing an Existing Entry

```bash
# Update a stale or incorrect ARP entry
sudo ip neigh change 192.168.1.50 lladdr 11:22:33:44:55:66 dev eth0
```

## Deleting an ARP Entry

```bash
# Delete a specific entry
sudo ip neigh del 192.168.1.50 dev eth0

# Flush all entries for an interface
sudo ip neigh flush dev eth0

# Flush only STALE entries
sudo ip neigh flush dev eth0 nud stale
```

## Forcing ARP Resolution

To immediately trigger ARP and populate the cache:

```bash
# Ping once to trigger ARP resolution
ping -c 1 192.168.1.50

# Now view the resulting cache entry
ip neigh show 192.168.1.50
```

## Checking for ARP Conflicts

If two hosts claim the same IP, you may see an ARP entry flip between two MACs:

```bash
# Monitor ARP activity in real time
sudo tcpdump -i eth0 -n arp &

# Watch for duplicate IP warnings in the kernel log
sudo journalctl -k | grep -i "duplicate"
```

## ARP Cache Tuning

```bash
# Increase the ARP cache size (useful on busy /16 subnets)
sudo sysctl -w net.ipv4.neigh.default.gc_thresh3=8192

# Make entries REACHABLE for longer (milliseconds)
sudo sysctl -w net.ipv4.neigh.eth0.base_reachable_time_ms=60000
```

## Conclusion

`ip neigh show` is the primary tool for inspecting ARP cache state. Use NUD states to understand whether entries are fresh or stale, add permanent entries for critical infrastructure like gateways, and use `ip neigh flush` to force re-resolution when MAC address changes occur.
