# How to Set ARP Cache Timeouts on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Networking, ARP, Linux, Performance, Tuning

Description: Learn how to configure ARP cache timeouts on Linux using sysctl parameters to optimize ARP behavior for your network.

## Default ARP Cache Behavior

Linux uses a neighbor discovery (ND) subsystem that manages ARP cache entries through states. The key timing parameters are controlled via `sysctl`:

```bash
# View all ARP/neighbor timing settings

sysctl -a | grep -E 'neigh.default\.(base_reachable|gc_stale|delay|retrans|gc_thresh)'
```

Typical defaults:

```text
net.ipv4.neigh.default.base_reachable_time_ms = 30000    # 30 seconds
net.ipv4.neigh.default.gc_stale_time = 60                # 60 seconds
net.ipv4.neigh.default.delay_first_probe_time = 5        # 5 seconds
net.ipv4.neigh.default.retrans_time_ms = 1000            # 1 second
net.ipv4.neigh.default.ucast_solicit = 3                 # 3 unicast probes
net.ipv4.neigh.default.mcast_solicit = 3                 # 3 multicast probes
```

## State Lifecycle and Timeouts

```text
NEW → INCOMPLETE → REACHABLE → STALE → DELAY → PROBE → FAILED
                      ↑ base_reachable_time_ms
                               ↑ gc_stale_time
                                         ↑ delay_first_probe_time
                                                  ↑ retrans_time × ucast_solicit
```

## Increasing Reachable Time (Fewer Re-ARPs)

For stable networks where MACs rarely change, increase the reachable time:

```bash
# Increase REACHABLE state to 5 minutes
sudo sysctl -w net.ipv4.neigh.default.base_reachable_time_ms=300000

# Increase STALE retention to 5 minutes
sudo sysctl -w net.ipv4.neigh.default.gc_stale_time=300
```

This reduces ARP traffic on large networks with stable MAC assignments.

## Decreasing Timeouts (Faster Failover)

For environments where MAC addresses change frequently (e.g., containers, VMs with live migration):

```bash
# Faster detection of changed MACs
sudo sysctl -w net.ipv4.neigh.default.base_reachable_time_ms=5000
sudo sysctl -w net.ipv4.neigh.default.gc_stale_time=15
sudo sysctl -w net.ipv4.neigh.default.delay_first_probe_time=2
sudo sysctl -w net.ipv4.neigh.default.retrans_time_ms=500
sudo sysctl -w net.ipv4.neigh.default.ucast_solicit=2
```

## Tuning ARP Table Size for Large Environments

On routers, hypervisors, or servers with many neighbors:

```bash
# Check current limits
sysctl net.ipv4.neigh.default.gc_thresh1
sysctl net.ipv4.neigh.default.gc_thresh2
sysctl net.ipv4.neigh.default.gc_thresh3

# Increase for large networks
sudo sysctl -w net.ipv4.neigh.default.gc_thresh1=512
sudo sysctl -w net.ipv4.neigh.default.gc_thresh2=2048
sudo sysctl -w net.ipv4.neigh.default.gc_thresh3=4096
```

| Parameter | Default | Meaning |
|-----------|---------|---------|
| gc_thresh1 | 128 | GC won't run below this (entries safe) |
| gc_thresh2 | 512 | GC starts if this is exceeded for 5+ seconds |
| gc_thresh3 | 1024 | Hard limit; GC runs immediately at this point |

## Per-Interface Timeout Configuration

```bash
# Set timeout for eth0 specifically
sudo sysctl -w net.ipv4.neigh.eth0.base_reachable_time_ms=60000
sudo sysctl -w net.ipv4.neigh.eth0.gc_stale_time=120
```

## Making Changes Persistent

```bash
# Add to /etc/sysctl.conf
cat >> /etc/sysctl.conf << 'EOF'
# ARP cache tuning
net.ipv4.neigh.default.base_reachable_time_ms = 300000
net.ipv4.neigh.default.gc_stale_time = 300
net.ipv4.neigh.default.gc_thresh1 = 512
net.ipv4.neigh.default.gc_thresh2 = 2048
net.ipv4.neigh.default.gc_thresh3 = 4096
EOF

sudo sysctl -p
```

## Verifying Current Entry Ages

```bash
# Watch entry states over time
watch -n 2 'ip neigh show | awk "{print \$1, \$NF}"'
```

## Key Takeaways

- `base_reachable_time_ms` controls how long REACHABLE entries last (default 30s).
- `gc_stale_time` controls how long STALE entries are kept before probing (default 60s).
- Increase timeouts in stable environments to reduce ARP traffic.
- Increase `gc_thresh` values on systems with many neighbors to prevent cache overflow.

**Related Reading:**

- [How to Understand ARP Cache Timeout and Expiration](https://oneuptime.com/blog/post/2026-03-20-arp-cache-timeout-expiration/view)
- [How to View the ARP Table on Linux](https://oneuptime.com/blog/post/2026-03-20-view-arp-table-linux/view)
- [How to Clear the ARP Cache](https://oneuptime.com/blog/post/2026-03-20-clear-arp-cache-linux/view)
