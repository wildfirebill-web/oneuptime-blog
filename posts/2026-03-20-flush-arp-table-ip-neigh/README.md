# How to Flush the ARP Table with ip neigh flush

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, ip command, iproute2, ARP, Networking, Troubleshooting

Description: Flush and clear the ARP table on Linux using ip neigh flush to force ARP re-resolution, useful for troubleshooting stale entries and IP change scenarios.

## Introduction

Flushing the ARP table forces the kernel to re-resolve MAC addresses for all neighbors. This is needed when IP-to-MAC mappings have changed (e.g., a NIC was replaced, a VM migrated, or ARP poisoning occurred). `ip neigh flush` removes dynamic entries while preserving permanent ones.

## Flush All Dynamic ARP Entries

```bash
# Flush all non-permanent neighbor entries

ip neigh flush all

# Verify the cache is mostly empty
ip neigh show
```

## Flush ARP Table for a Specific Interface

```bash
# Flush only eth0's ARP entries
ip neigh flush dev eth0
```

## Flush Only Stale Entries

```bash
# Remove only stale (expired) entries
ip neigh flush nud stale

# Remove failed entries
ip neigh flush nud failed
```

## Flush and Verify

```bash
# Before flush - see current entries
ip neigh show

# Flush
ip neigh flush dev eth0

# After flush - most entries removed
ip neigh show

# Generate new ARP entries by pinging
ping -c 1 192.168.1.1
ip neigh show 192.168.1.1
```

## Flush IPv4 ARP Only

```bash
# Flush only IPv4 neighbor entries
ip -4 neigh flush dev eth0
```

## Flush Everything Including Permanent (Use Caution)

```bash
# Delete all entries including permanent ones
ip neigh flush all nud all

# This includes static entries - use with care
```

## When to Flush the ARP Cache

- After changing a network interface card (MAC address changed)
- After migrating a VM to a different host
- When a device changed its IP without ARP announcement
- When troubleshooting "IP address already in use" errors
- After detecting ARP poisoning/spoofing

## Flush ARP on Remote Hosts Simultaneously

```bash
# If multiple hosts have stale ARP entries
# Use a loop to flush on all servers via SSH
for host in server1 server2 server3; do
    ssh $host "ip neigh flush dev eth0"
done
```

## Difference from Delete

```bash
# ip neigh flush - removes all matching entries at once
ip neigh flush dev eth0

# ip neigh del - removes a specific single entry
ip neigh del 192.168.1.50 dev eth0
```

## Conclusion

`ip neigh flush dev <interface>` clears all dynamic ARP entries for an interface, forcing re-resolution. Use `nud stale` or `nud failed` to flush only specific states. Permanent entries survive flush unless `nud all` is specified. After flushing, new ARP entries are populated as traffic flows.
