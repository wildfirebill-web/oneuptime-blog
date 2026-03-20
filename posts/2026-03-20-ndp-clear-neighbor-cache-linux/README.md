# How to Clear the IPv6 Neighbor Cache on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: NDP, Neighbor Cache, Linux, ip command, Troubleshooting

Description: Clear IPv6 neighbor cache entries on Linux to force re-resolution, remove stale entries, and troubleshoot connectivity issues caused by cached stale MAC addresses.

## Introduction

Clearing the IPv6 neighbor cache forces re-resolution of IPv6 addresses to MAC addresses via NDP. This is necessary when a host's MAC address changes (NIC replacement, virtualization migration), when connectivity fails due to a stale neighbor cache entry, or when troubleshooting NDP issues. The `ip -6 neigh` command provides granular control over cache entry management.

## Clearing Neighbor Cache Entries

```bash
# Delete a single neighbor entry
sudo ip -6 neigh del 2001:db8::1 dev eth0

# Delete the link-local entry for a neighbor
sudo ip -6 neigh del fe80::1 dev eth0

# Flush all neighbor cache entries for an interface
sudo ip -6 neigh flush dev eth0

# Flush all entries across all interfaces
sudo ip -6 neigh flush all

# Flush only STALE entries (leaves REACHABLE and PERMANENT)
sudo ip -6 neigh flush nud stale

# Flush only FAILED entries
sudo ip -6 neigh flush nud failed

# Flush STALE and FAILED entries together
sudo ip -6 neigh flush nud stale dev eth0
sudo ip -6 neigh flush nud failed dev eth0

# Verify the flush was effective
ip -6 neigh show dev eth0
# Should be empty or show only recently re-resolved entries
```

## When to Clear the Neighbor Cache

```
Scenarios requiring neighbor cache clearing:

1. MAC address change (host replaced NIC or VM migrated):
   sudo ip -6 neigh del <old_ip> dev eth0
   → Forces re-ARP/NDP to discover new MAC
   → Or wait for REACHABLE timer to expire naturally (30s)

2. "Works for small packets, fails for large" (PMTU issue):
   Not a neighbor cache issue, but often confused with one
   Actually needs: sudo ip -6 route flush cache (for PMTU cache)

3. Host moved to different switch port (different MAC path):
   sudo ip -6 neigh flush dev eth0
   → All neighbors re-resolved

4. Debugging NDP behavior:
   sudo ip -6 neigh flush dev eth0
   → Clean state for capturing fresh NS/NA exchange

5. After network reconfiguration:
   sudo ip -6 neigh flush all
   → Ensures no stale entries from old network topology
```

## Removing Specific Entry Types

```bash
# Show only STALE entries before clearing
ip -6 neigh show nud stale

# Show only FAILED entries
ip -6 neigh show nud failed

# Remove all entries except PERMANENT (static) ones
ip -6 neigh show | grep -v PERMANENT | while read addr rest; do
    dev=$(echo "$rest" | grep -oP '(?<=dev )\S+')
    if [ -n "$dev" ]; then
        sudo ip -6 neigh del "$addr" dev "$dev" 2>/dev/null
    fi
done

# Add a static (permanent) entry that won't be cleared
sudo ip -6 neigh add 2001:db8::1 lladdr 00:11:22:33:44:55 dev eth0
# Permanent entries persist across cache flushes

# View permanent entries
ip -6 neigh show nud permanent
```

## Triggering Re-Resolution After Clear

```bash
# After clearing, trigger re-resolution by sending traffic
# The kernel will automatically send NS to re-resolve addresses

# Force immediate re-resolution by pinging
ping6 -c 1 2001:db8::1
# This triggers an NS → NA exchange and populates the cache

# Watch the NS being sent after clearing
sudo ip -6 neigh flush dev eth0
sudo tcpdump -i eth0 -v "icmp6 and ip6[40] == 135" -c 3 &
ping6 -c 1 2001:db8::1  # Triggers NS

# Check the cache is populated after re-resolution
sleep 1
ip -6 neigh show | grep 2001:db8::1
```

## Scripted Cache Management

```bash
#!/bin/bash
# Clear IPv6 neighbor cache and report results

INTERFACE="${1:-eth0}"

echo "=== Before flush ==="
ip -6 neigh show dev "$INTERFACE" | wc -l
echo " neighbor entries"

echo ""
echo "Flushing neighbor cache for $INTERFACE..."
sudo ip -6 neigh flush dev "$INTERFACE"

echo ""
echo "=== After flush ==="
ip -6 neigh show dev "$INTERFACE" | wc -l
echo " neighbor entries remaining (should be 0 or minimal)"

echo ""
echo "Waiting for re-resolution..."
sleep 2
ip -6 neigh show dev "$INTERFACE"
```

## Conclusion

The `ip -6 neigh flush` command is the standard tool for clearing IPv6 neighbor cache entries on Linux. Specific entries can be deleted with `ip -6 neigh del`, while `flush` removes groups of entries by interface or NUD state. Clearing the cache is most useful when a MAC address has changed and the stale cache entry is causing connectivity failure. For most transient connectivity issues, it is better to wait for the NUD state machine to handle re-resolution automatically through the STALE → DELAY → PROBE sequence.
