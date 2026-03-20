# How to Use tracepath to Discover Path MTU on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: tracepath, MTU, PMTUD, Linux, Networking, Troubleshooting

Description: Use tracepath to discover the path MTU between two hosts, interpret its output to find MTU bottlenecks, and compare tracepath with traceroute and ping for MTU discovery.

## Introduction

`tracepath` is a Linux utility that combines traceroute functionality with Path MTU Discovery. It sends packets with increasing TTL values while simultaneously probing the MTU at each hop. Unlike `traceroute`, `tracepath` requires no root privileges and shows the MTU reduction points along the path. This makes it the first tool to reach for when diagnosing MTU and fragmentation issues.

## Basic tracepath Usage

```bash
# Trace path to destination showing MTU:
tracepath 10.20.0.5

# With numeric output (no DNS resolution):
tracepath -n 10.20.0.5

# IPv6 path MTU:
tracepath6 -n 2001:db8::1

# Example output:
# 1?: [LOCALHOST]                     pmtu 1500
# 1:  192.168.1.1                     0.311ms
# 1:  192.168.1.1                     0.257ms
# 2:  10.0.0.1                        1.234ms
# 3:  203.0.113.1                     5.678ms  asymm  2
# 4:  10.20.0.5                       6.123ms reached
#     Resume: pmtu 1500 hops 4 back 4

# "pmtu" shows the discovered path MTU
# "asymm" means reverse path is different
```

## Interpret tracepath Output

```
tracepath output fields:

1?: [LOCALHOST]   pmtu 1500
│   │              └─ path MTU discovered so far
│   └─ local machine
└─ hop number

10.0.0.1          1.234ms  asymm  2
│                  │         │     └─ reverse TTL (asymmetric route if different)
│                  │         └─ asymmetric routing detected
│                  └─ round trip time
└─ router IP address

* * *            No response (ICMP not returned or hop silent)

Resume: pmtu 1480 hops 6 back 6
         │           │      └─ TTL in return packets (should match hops for symmetric)
         │           └─ number of hops
         └─ final path MTU discovered

If pmtu decreases mid-path:
  The hop where it decreases is the MTU bottleneck
```

## Find MTU Bottleneck

```bash
# Run tracepath and look for MTU changes:
tracepath -n 203.0.113.50

# Output showing MTU reduction:
# 1?: [LOCALHOST]                     pmtu 1500
# 1:  10.0.0.1                        0.3ms
# 2:  192.0.2.1                       2.1ms
# 3:  203.0.113.1                     5.4ms   pmtu 1480
# 4:  203.0.113.50                    5.9ms   reached
#     Resume: pmtu 1480 hops 4 back 4

# Hop 3 at 203.0.113.1 is where MTU drops to 1480
# This is your bottleneck router

# If you see pmtu drop at a tunnel entry:
# 3:  10.200.0.1                     1.2ms   pmtu 1476
# Indicates GRE tunnel with 24-byte overhead

# VXLAN tunnel:
# 2:  10.100.0.1                     0.8ms   pmtu 1450
# Indicates VXLAN with 50-byte overhead
```

## Compare tracepath with Other Tools

```bash
# tracepath: automatic MTU discovery, no root needed
tracepath -n 10.20.0.5

# ping with DF bit: manual MTU probing
ping -M do -s 1472 -c 1 10.20.0.5  # Tests exactly 1500 bytes
ping -M do -s 1452 -c 1 10.20.0.5  # Tests exactly 1480 bytes

# traceroute: shows hops but not MTU
traceroute -n 10.20.0.5

# mtr: real-time traceroute, no built-in MTU
mtr -n 10.20.0.5

# For complete MTU discovery:
# tracepath gives you the pmtu
# Then verify with ping -M do at that size
ping -M do -s $(($(tracepath -n 10.20.0.5 | grep 'pmtu' | \
  grep -oP 'pmtu \K[0-9]+' | tail -1) - 28)) -c 3 10.20.0.5
```

## Script: MTU Discovery Along Path

```bash
#!/bin/bash
# Discover MTU at each tracepath hop

DEST=$1
echo "Path MTU Discovery to $DEST"
echo "================================"

# Run tracepath and parse pmtu values:
tracepath -n "$DEST" 2>/dev/null | while IFS= read -r line; do
    HOP=$(echo "$line" | awk '{print $1}')
    IP=$(echo "$line" | grep -oP '\d+\.\d+\.\d+\.\d+' | head -1)
    PMTU=$(echo "$line" | grep -oP 'pmtu \K[0-9]+')

    if [ -n "$PMTU" ]; then
        echo "Hop $HOP ($IP): MTU change detected -> $PMTU bytes"
    fi

    if echo "$line" | grep -q "reached"; then
        FINAL_PMTU=$(echo "$line" | grep -oP 'pmtu \K[0-9]+')
        # Final from Resume line
    fi
done

# Get final pmtu from Resume line:
FINAL=$(tracepath -n "$DEST" 2>/dev/null | grep "Resume" | \
  grep -oP 'pmtu \K[0-9]+')
echo ""
echo "Final Path MTU: ${FINAL:-unknown} bytes"
echo "Recommended TCP MSS: $((${FINAL:-1500} - 40)) bytes"
```

## tracepath Limitations and Workarounds

```bash
# Limitation 1: Some routers don't respond to UDP probes
# Workaround: use ping with DF bit instead
ping -M do -s 1472 -c 3 10.20.0.5

# Limitation 2: Firewalls may block ICMP Fragmentation Needed
# Workaround: test from both ends; check if black hole exists
tracepath -n 10.20.0.5
# If no pmtu changes but connections fail: MTU black hole

# Limitation 3: IPv6 path MTU behaves differently
# IPv6 never fragments in transit; all fragmentation is end-to-end
tracepath6 -n 2001:db8::1

# Limitation 4: Asymmetric paths
# tracepath only discovers forward path MTU
# Return path may have different MTU
# Test from destination back to source if possible
```

## Conclusion

`tracepath -n <destination>` is the fastest way to discover path MTU on Linux — it requires no root privileges and shows MTU at each hop. Look for `pmtu` values that decrease along the path to identify bottleneck routers. The final `Resume: pmtu N` line shows the effective path MTU. Use this value to configure application payload sizes, tunnel interface MTU, and TCP MSS clamping rules. For hops that don't respond, verify with `ping -M do` at specific sizes to confirm the MTU boundary.
