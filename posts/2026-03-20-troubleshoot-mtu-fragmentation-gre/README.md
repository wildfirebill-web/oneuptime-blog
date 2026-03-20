# How to Troubleshoot MTU and Fragmentation in GRE Tunnels

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, GRE, Tunnel, MTU, Fragmentation, Networking, Troubleshooting, TCP MSS

Description: Identify and fix MTU fragmentation issues in GRE tunnels where large packets fail but small packets succeed, using MSS clamping and MTU adjustments.

## Introduction

A classic GRE tunnel symptom: small pings work but large TCP transfers fail or large files stall. This is an MTU/fragmentation issue. GRE adds 24 bytes of overhead, reducing the usable payload from 1500 to 1476 bytes. If the DF (Don't Fragment) bit is set and the packet exceeds 1476 bytes, it is silently dropped.

## Understanding the Issue

```
Physical MTU:      1500 bytes
GRE overhead:       -24 bytes (20 IP + 4 GRE)
Tunnel MTU:        1476 bytes
TCP MSS (without clamping): 1460 bytes → too large!
TCP MSS (correct): 1436 bytes (1476 - 40 for IP+TCP headers)
```

## Diagnose the Problem

```bash
# Small ping succeeds
ping -s 100 -M do -c 3 192.168.2.1

# Large ping fails with "Message too long" or no reply
ping -s 1453 -M do -c 3 192.168.2.1
# ping: local error: message too long, mtu=1476
```

## Fix 1: Set the Correct MTU on the Tunnel Interface

```bash
# Set GRE tunnel MTU to 1476
ip link set gre0 mtu 1476

# Verify
ip link show gre0 | grep mtu
```

## Fix 2: TCP MSS Clamping (Most Effective)

MSS clamping modifies TCP SYN packets to advertise a smaller MSS, preventing large TCP segments:

```bash
# Clamp MSS automatically to path MTU
iptables -A FORWARD -o gre0 -p tcp --tcp-flags SYN,RST SYN \
    -j TCPMSS --clamp-mss-to-pmtu

# Or clamp to an explicit value
iptables -A FORWARD -o gre0 -p tcp --tcp-flags SYN,RST SYN \
    -j TCPMSS --set-mss 1436

# Also clamp for traffic coming back through the tunnel
iptables -A FORWARD -i gre0 -p tcp --tcp-flags SYN,RST SYN \
    -j TCPMSS --clamp-mss-to-pmtu
```

## Fix 3: nftables MSS Clamping

```bash
# Clamp MSS with nftables
nft add rule inet filter forward \
    oif "gre0" tcp flags syn tcp option maxseg size set rt mtu
```

## Test After Fix

```bash
# After applying MSS clamping, test large transfers
# 1: Test large ICMP (should work after MTU fix)
ping -s 1453 -M do -c 3 192.168.2.1

# 2: Test a file transfer (should no longer stall)
curl -o /dev/null http://192.168.2.10/largefile

# 3: Verify MSS in TCP handshake with tcpdump
tcpdump -i gre0 "tcp[tcpflags] & tcp-syn != 0" -n
# Look for MSS option: "mss 1436" or similar
```

## Layered Overhead for Different Tunnel Types

```
GRE:          1476 (1500 - 24)
GRE+IPsec:    1422 (1500 - 24 GRE - 54 IPsec)
VXLAN:        1450 (1500 - 50 VXLAN overhead)
GRE over IPv6: 1456 (1500 - 44)
```

## Conclusion

GRE MTU/fragmentation issues are caused by the 24-byte overhead reducing the effective payload. The best fix is TCP MSS clamping using `--clamp-mss-to-pmtu` on the FORWARD chain for the tunnel interface, which handles MSS negotiation automatically. Also set the tunnel interface MTU to 1476 so the OS correctly reports the tunnel's capability. Large ICMP failures with DF bit set confirm the issue.
