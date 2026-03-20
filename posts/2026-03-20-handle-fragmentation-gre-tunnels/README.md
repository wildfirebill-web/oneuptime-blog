# How to Handle Fragmentation with GRE Tunnels

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GRE, Fragmentation, MTU, Tunneling, Linux, Networking

Description: Understand GRE tunnel overhead, configure correct MTU to prevent fragmentation, and apply TCP MSS clamping to ensure reliable traffic through GRE tunnels.

## Introduction

GRE (Generic Routing Encapsulation) adds 24 bytes of overhead to each packet: a 20-byte outer IP header and a 4-byte GRE header. When the physical interface MTU is 1500 bytes, the maximum payload MTU through a GRE tunnel is 1476 bytes. Without correct MTU configuration, packets are fragmented at the tunnel entry point or silently dropped if DF bit is set.

## GRE Overhead Breakdown

```
GRE Tunnel Overhead:
  Outer IP header:  20 bytes
  GRE header:        4 bytes (basic GRE without optional fields)
  Total overhead:   24 bytes

  Physical MTU 1500:
    GRE tunnel MTU = 1500 - 24 = 1476 bytes

  With GRE + IPsec (transport mode):
    1500 - 24 (GRE) - 58 (IPsec) = 1418 bytes

  With GRE + optional checksum/key:
    1500 - 28 (GRE with key) = 1472 bytes

For TCP through GRE tunnel:
  GRE MTU 1476 - 40 (TCP+IP headers) = MSS 1436
```

## Create GRE Tunnel with Correct MTU

```bash
# Create GRE tunnel:
ip tunnel add gre1 mode gre local 192.168.1.1 remote 203.0.113.1 ttl 255

# Set MTU accounting for GRE overhead:
ip link set gre1 mtu 1476

# Bring up and assign IP:
ip link set gre1 up
ip addr add 10.200.0.1/30 dev gre1

# Add route through tunnel:
ip route add 10.100.0.0/16 dev gre1

# Verify MTU:
ip link show gre1
# Should show: mtu 1476
```

## Apply TCP MSS Clamping on GRE Tunnel

```bash
# MSS clamping prevents oversized TCP segments:

# For traffic entering the tunnel (FORWARD chain):
iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN \
  -o gre1 -j TCPMSS --set-mss 1436

iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN \
  -i gre1 -j TCPMSS --set-mss 1436

# For traffic originating from this host:
iptables -t mangle -A OUTPUT -p tcp --tcp-flags SYN,RST SYN \
  -o gre1 -j TCPMSS --set-mss 1436

# Verify iptables rules:
iptables -t mangle -L FORWARD -v -n | grep TCPMSS

# Alternative: use --clamp-mss-to-pmtu (auto-detects PMTU):
iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN \
  -o gre1 -j TCPMSS --clamp-mss-to-pmtu
```

## Handle DF Bit in GRE Encapsulation

```bash
# Problem: inner packet has DF set, outer packet inherits DF
# When outer packet exceeds physical MTU: ICMP Fragmentation Needed generated

# Check if DF propagation is enabled (Linux default: enabled):
cat /proc/sys/net/ipv4/ip_no_pmtu_disc
# 0 = PMTUD enabled (DF bit propagated)
# 1 = PMTUD disabled (DF bit cleared, fragmentation allowed)

# View GRE tunnel DF settings:
ip -d link show gre1 | grep nopmtudisc

# Allow outer fragmentation (clears DF on outer header):
ip tunnel change gre1 nopmtudisc

# This allows the outer GRE packet to be fragmented
# at the cost of more fragmentation events

# Better: keep PMTUD enabled, fix MTU and MSS clamping
ip tunnel change gre1 pmtudisc  # Default, keeps DF propagation
```

## Test Fragmentation Through GRE Tunnel

```bash
# Test with progressively larger packets:
for SIZE in 1500 1476 1450 1400; do
    PAYLOAD=$((SIZE - 28))
    RESULT=$(ping -M do -s $PAYLOAD -c 1 -W 2 10.100.0.1 2>&1)
    if echo "$RESULT" | grep -q "bytes from"; then
        echo "MTU $SIZE through GRE: PASS"
    else
        echo "MTU $SIZE through GRE: FAIL"
    fi
done

# Check for fragmentation activity:
watch -n 1 "nstat | grep -E 'IpFrag|IpReasm'"

# Capture fragments on GRE tunnel interface:
tcpdump -i gre1 -n 'ip[6:2] & 0x3fff != 0'
# Shows packets with MF bit or non-zero fragment offset
```

## Persistent GRE Tunnel with Correct MTU

```bash
# Using systemd-networkd:
cat > /etc/systemd/network/30-gre1.netdev << 'EOF'
[NetDev]
Name=gre1
Kind=gre

[Tunnel]
Local=192.168.1.1
Remote=203.0.113.1
TTL=255
EOF

cat > /etc/systemd/network/30-gre1.network << 'EOF'
[Match]
Name=gre1

[Link]
MTUBytes=1476

[Network]
Address=10.200.0.1/30
EOF

networkctl reload

# Or via /etc/network/interfaces:
# auto gre1
# iface gre1 inet static
#     address 10.200.0.1
#     netmask 255.255.255.252
#     pre-up ip tunnel add gre1 mode gre local 192.168.1.1 remote 203.0.113.1
#     up ip link set gre1 mtu 1476
#     post-down ip tunnel del gre1
```

## Conclusion

GRE tunnels require MTU set to physical-MTU minus 24 bytes (typically 1476 for 1500-byte physical). Always apply TCP MSS clamping on the tunnel interface to prevent TCP sessions from sending segments too large for the tunnel. If you have GRE over IPsec, subtract both overheads: 24 (GRE) + ~58 (IPsec) = 82 bytes total, leaving 1418 bytes. Monitor `IpFragCreates` and `IpReasmFails` to detect ongoing fragmentation issues through the tunnel.
