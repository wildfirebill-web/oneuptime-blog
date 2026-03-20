# How to Remove All tc qdisc Rules from an IPv4 Interface

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: tc, Traffic Control, Qdisc, IPv4, Linux, Networking, Cleanup

Description: Cleanly remove all tc (traffic control) queuing discipline rules from a Linux network interface to restore default behavior for IPv4 traffic.

## Introduction

After applying `tc` queuing disciplines for bandwidth limiting, traffic shaping, or network simulation, you may need to remove all rules to restore normal behavior. This guide covers how to cleanly remove root qdiscs, ingress qdiscs, and verify the interface has returned to its default state.

## Understanding tc qdisc Hierarchy

Linux network interfaces have two attachment points for qdiscs:
- **Root (egress)**: Controls outbound traffic - attached with `tc qdisc add dev ethX root ...`
- **Ingress**: Controls inbound traffic - attached with `tc qdisc add dev ethX ingress`

Each must be removed separately.

## Removing the Root (Egress) qdisc

```bash
# Remove the root qdisc from eth0 (removes all child classes and filters too)

sudo tc qdisc del dev eth0 root

# Verify it's gone
sudo tc qdisc show dev eth0
# Should show only: qdisc noqueue or qdisc fq_codel (the default)
```

## Removing the Ingress qdisc

```bash
# Remove the ingress qdisc from eth0
sudo tc qdisc del dev eth0 ingress

# Alternative form (handle ffff:)
sudo tc qdisc del dev eth0 handle ffff: ingress
```

## Removing All qdiscs from All Interfaces

For a full cleanup across all interfaces:

```bash
#!/bin/bash
# cleanup-tc.sh - Remove all custom tc rules from all interfaces

for iface in $(ip link show | awk -F': ' '/^[0-9]+: /{print $2}' | cut -d@ -f1); do
    # Skip loopback
    [[ "$iface" == "lo" ]] && continue
    
    # Remove root qdisc (suppress error if it doesn't exist)
    sudo tc qdisc del dev "${iface}" root 2>/dev/null && \
        echo "Removed root qdisc from ${iface}" || \
        echo "No root qdisc on ${iface}"
    
    # Remove ingress qdisc
    sudo tc qdisc del dev "${iface}" ingress 2>/dev/null && \
        echo "Removed ingress qdisc from ${iface}" || \
        echo "No ingress qdisc on ${iface}"
done
```

## Removing IFB Interfaces Used for Ingress Limiting

If you created IFB (Intermediate Functional Block) interfaces for ingress shaping:

```bash
# Remove tc rules on the IFB interface
sudo tc qdisc del dev ifb0 root 2>/dev/null

# Bring down and remove the IFB interface
sudo ip link set ifb0 down
sudo ip link del ifb0

# Unload the IFB kernel module if no longer needed
sudo modprobe -r ifb
```

## Verifying Complete Cleanup

```bash
# Check all qdiscs on eth0 - should show default system qdisc only
sudo tc qdisc show dev eth0

# Check all classes (should be empty if root qdisc was removed)
sudo tc class show dev eth0

# Check all filters (should be empty)
sudo tc filter show dev eth0
```

## Default qdisc After Removal

After removing custom qdiscs, the interface reverts to the system default:

```bash
# Show system-wide default qdisc
cat /proc/sys/net/core/default_qdisc
# Typically: fq_codel (Ubuntu/Debian), pfifo_fast (older systems)
```

## Persisting Cleanup Across Reboots

tc rules are transient - they disappear on reboot. However, if you have a startup script that applies tc rules, remove those entries or add the cleanup script to run on startup before the rules are applied.

```bash
# Remove tc rules from the startup script
sudo systemctl disable tc-rules.service

# Or add the cleanup to rc.local / network-up hook
```

## Conclusion

Removing `tc` qdiscs is straightforward - `tc qdisc del dev ethX root` removes all egress rules in one command, and `tc qdisc del dev ethX ingress` clears ingress redirect rules. Always verify with `tc qdisc show` to confirm the interface is clean.
