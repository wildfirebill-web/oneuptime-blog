# How to Find the Path MTU Between Two Hosts

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MTU, PMTUD, Networking, Linux, tracepath, Troubleshooting

Description: Discover the Path MTU (PMTU) between two hosts using tracepath, ping with DF bit, and socket-level PMTU discovery to determine the maximum packet size.

## Introduction

Path MTU is the smallest MTU of any link on the path between two hosts. Knowing the path MTU is essential for configuring applications, VPN overlays, and protocol encapsulation. Unlike interface MTU (which you can read directly), path MTU must be discovered dynamically since it depends on the complete network path which may include links with various MTUs.

## Method 1: tracepath

```bash
# tracepath automatically discovers MTU at each hop:
tracepath -n 10.20.0.5
# Output:
# 1: 10.20.0.1          0.5ms
# 2: 10.1.0.1          2.1ms  pmtu 9000  ← 9000 MTU (jumbo frames)
# 3: 203.0.113.1        5.2ms  pmtu 1500  ← bottleneck! MTU reduces to 1500
# 4: 10.20.0.5          7.8ms  reached

# The minimum pmtu in the path = path MTU
# Here: path MTU = 1500

# tracepath6 for IPv6:
tracepath6 -n 2001:db8::1
```

## Method 2: ping with DF Bit (Binary Search)

```bash
#!/bin/bash
# Find exact path MTU by binary search with ping

DEST="10.20.0.5"
LOW=576      # Minimum IPv4 MTU per RFC
HIGH=9000    # Maximum (jumbo frames)

echo "Finding path MTU to $DEST..."

while [ $((HIGH - LOW)) -gt 1 ]; do
    MID=$(( (LOW + HIGH) / 2 ))
    PAYLOAD=$((MID - 28))  # Subtract IP header (20) + ICMP header (8)

    if ping -M do -s $PAYLOAD -c 1 -W 2 $DEST > /dev/null 2>&1; then
        LOW=$MID
    else
        HIGH=$MID
    fi
done

echo "Path MTU: ${HIGH} bytes"
echo "Maximum payload without fragmentation:"
echo "  ICMP/ping: $((HIGH - 28)) bytes"
echo "  UDP:       $((HIGH - 28)) bytes"
echo "  TCP:       $((HIGH - 40)) bytes (MSS)"
```

## Method 3: Socket-Level PMTUD

```python
#!/usr/bin/env python3
# Discover path MTU using socket PMTUD

import socket

def get_path_mtu(destination, port=9):
    """Get path MTU using IP_PMTUDISC_PROBE."""
    IP_PMTUDISC_PROBE = 3

    sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    sock.setsockopt(socket.IPPROTO_IP, socket.IP_MTU_DISCOVER, IP_PMTUDISC_PROBE)
    sock.settimeout(3)

    try:
        sock.connect((destination, port))
        # Send probe packets to trigger PMTUD
        for size in [1472, 1450, 1400, 1300]:
            try:
                sock.send(b'X' * size)
            except OSError:
                pass

        # Read discovered MTU:
        mtu = sock.getsockopt(socket.IPPROTO_IP, socket.IP_MTU)
        return mtu
    except Exception as e:
        print(f"Error: {e}")
        return None
    finally:
        sock.close()

dest = "10.20.0.5"
mtu = get_path_mtu(dest)
if mtu:
    print(f"Path MTU to {dest}: {mtu} bytes")
    print(f"Max UDP payload:    {mtu - 28} bytes")
    print(f"TCP MSS:            {mtu - 40} bytes")
```

## Common Path MTU Values

```
Network Type             | Typical Path MTU
-------------------------|------------------
Standard Ethernet        | 1500 bytes
Jumbo frames (LAN)       | 9000 bytes
PPPoE (DSL/cable)        | 1492 bytes
GRE tunnel               | 1476 bytes (1500 - 24)
IPsec ESP (AES-256-GCM)  | ~1422 bytes (varies with cipher/mode)
VXLAN                    | 1450 bytes (1500 - 50)
WireGuard VPN            | 1420 bytes (1500 - 80)
WireGuard in VXLAN       | 1370 bytes
Cellular/Mobile          | 1400-1500 bytes (variable)
Satellite                | 576-1500 bytes (variable)
```

## Verify with Direct Test

```bash
# Verify the discovered path MTU actually works:
PATH_MTU=1500  # Replace with your discovered MTU

# ICMP test at path MTU (should succeed):
ping -M do -s $((PATH_MTU - 28)) -c 3 10.20.0.5

# ICMP test 1 byte above path MTU (should fail):
ping -M do -s $((PATH_MTU - 27)) -c 1 10.20.0.5
# Should show: "Frag needed and DF set (mtu = XXXX)"

# TCP should automatically discover and use path MTU:
# Connect with iperf3 and check ss output:
iperf3 -c 10.20.0.5 -t 5 &
ss -tin state established dst 10.20.0.5 | grep mss
# MSS should be approximately PATH_MTU - 40
```

## Conclusion

Use `tracepath -n` for a quick path MTU discovery — it reports the MTU at each hop and shows where the bottleneck is. For programmatic use, the socket `IP_PMTUDISC_PROBE` option returns the kernel's current path MTU estimate after sending. TCP discovers path MTU automatically using PMTUD; configure VPN tunnels and application-level UDP protocols to use `path_mtu - overhead` to avoid fragmentation. The common problem environments are IPsec VPNs (50-80 bytes overhead) and VXLAN overlays (50 bytes overhead) where the effective path MTU is significantly less than the underlying Ethernet MTU.
