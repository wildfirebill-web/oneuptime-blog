# How to Use traceroute vs tracepath on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Traceroute, Tracepath, Linux, Networking, MTU, Diagnostics

Description: Compare traceroute and tracepath on Linux, understand when to use each, and leverage tracepath's unique ability to discover path MTU alongside hop information.

Both tools map network paths but work differently. `tracepath` doesn't require root privileges and automatically discovers Path MTU — making it ideal for quick diagnostics from non-privileged user accounts.

## Key Differences

```
Feature              traceroute           tracepath
-------------------  -------------------  -----------------------
Requires root?       Yes (for ICMP/TCP)   No (uses UDP unprivileged)
Protocol             UDP/ICMP/TCP         UDP (PMTUD probes)
MTU discovery        No                   Yes (built-in)
DNS resolution       Configurable         Yes (use -n to disable)
Multiple probes      3 per hop            1 per hop
Output format        Standard hops        Hops + MTU info
Installation         sudo apt install     Usually pre-installed
```

## Using tracepath

```bash
# Basic tracepath (no root needed)
tracepath google.com

# Skip DNS resolution (faster)
tracepath -n 8.8.8.8

# Example output:
# 1?: [LOCALHOST]                                         pmtu 1500
# 1:  192.168.1.1                                          1.2ms
# 1:  192.168.1.1                                          1.1ms
# 2:  10.1.0.1                                             8.3ms
# 3:  72.14.0.1                                           12.1ms asymm  4
# 4:  8.8.8.8                                             12.8ms reached
#     Resume: pmtu 1500 hops 4 back 4
```

## Reading tracepath MTU Discovery Output

```bash
tracepath -n 192.168.1.1

# Key fields:
# "pmtu 1500" — Path MTU discovered at this point in the path
# "asymm 4"   — Asymmetric route: forward hops ≠ return hops
# "reached"   — Destination responded

# If MTU changes:
# 1:  192.168.1.1    pmtu 1500
# 3:  10.0.0.1       pmtu 1492  ← MTU shrinks here (PPPoE link)
# This tells you exactly where the MTU reduction occurs
```

## Using traceroute

```bash
# Standard UDP traceroute
traceroute -n 8.8.8.8

# ICMP traceroute (works through more firewalls)
sudo traceroute -I -n 8.8.8.8

# TCP traceroute on port 80 (bypasses ICMP/UDP filters)
sudo traceroute -T -p 80 -n 8.8.8.8

# Set timeout per hop (default 5 seconds)
traceroute -w 2 -n 8.8.8.8

# Limit max hops
traceroute -m 15 -n 8.8.8.8
```

## When to Use Which Tool

```bash
# Use tracepath when:
#   - You don't have root access
#   - You also need to check path MTU
#   - Quick hop-count is all you need
tracepath -n 8.8.8.8

# Use traceroute when:
#   - You need more detail (3 probes per hop)
#   - You need to choose probe type (UDP/ICMP/TCP)
#   - You want to force a specific interface or source IP
sudo traceroute -I -n -w 1 8.8.8.8

# Use TCP traceroute when:
#   - Standard traceroute shows *** for many hops (firewall blocking)
sudo traceroute -T -p 443 -n 8.8.8.8
```

## Compare Outputs Side by Side

```bash
# Run both to compare hop counts and IPs
echo "=== traceroute ==="
traceroute -n -m 10 8.8.8.8

echo ""
echo "=== tracepath ==="
tracepath -n 8.8.8.8
```

## tracepath for MTU Debugging

tracepath's MTU discovery is its killer feature for VPN/tunnel troubleshooting:

```bash
# Check path MTU to a server behind a VPN
tracepath -n 10.200.0.1

# If output shows:
# 1:  10.8.0.1      pmtu 1500
# 3:  10.8.0.1      pmtu 1420  ← MTU reduced at tunnel entry
# This explains why large file transfers fail over the VPN

# Fix: set MTU on the VPN interface to match
sudo ip link set tun0 mtu 1420
```

Use `tracepath` as your first tool when you want quick path info without root, and `traceroute` when you need protocol control for bypassing firewalls or collecting multi-probe statistics.
