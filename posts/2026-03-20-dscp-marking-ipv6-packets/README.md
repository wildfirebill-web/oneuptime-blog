# How to Configure DSCP Marking for IPv6 Packets

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DSCP, IPv6, QoS, Traffic Marking, Ip6tables, nftables, Linux

Description: Configure DSCP marking for IPv6 packets on Linux using ip6tables, nftables, and tc to enable differentiated service treatment for different traffic types.

---

DSCP (Differentiated Services Code Point) marking sets priority bits in the IPv6 Traffic Class field to signal routers and switches how to handle packets. Properly marking IPv6 traffic enables QoS enforcement throughout the network path.

## DSCP Marking with ip6tables (mangle table)

```bash
# Basic DSCP marking rules for IPv6 using ip6tables

# Mark VoIP signaling (SIP) with CS5

sudo ip6tables -t mangle -A PREROUTING \
  -p tcp --dport 5060 \
  -j DSCP --set-dscp-class CS5

sudo ip6tables -t mangle -A PREROUTING \
  -p udp --dport 5060 \
  -j DSCP --set-dscp-class CS5

# Mark VoIP media (RTP) with EF
sudo ip6tables -t mangle -A PREROUTING \
  -p udp --dport 10000:20000 \
  -j DSCP --set-dscp 46

# Mark video conferencing with AF41
sudo ip6tables -t mangle -A PREROUTING \
  -p udp \
  -m owner --uid-owner zoom \
  -j DSCP --set-dscp-class AF41

# Mark interactive traffic (SSH) with AF31
sudo ip6tables -t mangle -A PREROUTING \
  -p tcp --dport 22 \
  -j DSCP --set-dscp-class AF31

# Mark bulk data with AF11 (low priority)
sudo ip6tables -t mangle -A PREROUTING \
  -p tcp --dport 8080:8090 \
  -j DSCP --set-dscp-class AF11

# Default: mark remaining traffic as best effort (CS0)
```

## DSCP Marking with nftables for IPv6

```bash
# /etc/nftables.conf - DSCP marking for IPv6

table ip6 mangle {
    chain prerouting {
        type filter hook prerouting priority mangle; policy accept;

        # VoIP signaling - CS5 (40)
        tcp dport 5060 ip6 dscp set cs5
        udp dport 5060 ip6 dscp set cs5

        # VoIP media - EF (46)
        udp dport 10000-20000 ip6 dscp set ef

        # Video streaming - CS3 (24)
        tcp dport { 1935, 8554 } ip6 dscp set cs3

        # Interactive - AF31 (26)
        tcp dport { 22, 23 } ip6 dscp set af31

        # HTTP/S - default best effort
        tcp dport { 80, 443 } ip6 dscp set cs0
    }

    chain postrouting {
        type filter hook postrouting priority mangle; policy accept;

        # Re-mark ICMP6 as highest priority (CS7)
        ip6 nexthdr icmpv6 ip6 dscp set cs7
    }
}
```

```bash
sudo nft -f /etc/nftables.conf
sudo nft list ruleset
```

## DSCP Marking with tc (Traffic Control)

```bash
# Use tc to mark IPv6 traffic egressing an interface

# Install iproute2
sudo apt install iproute2 -y

# Add dsmark qdisc for DSCP marking
sudo tc qdisc add dev eth0 root dsmark \
  indices 64 default_index 0

# Mark IPv6 traffic with DSCP EF (46)
# Using u32 filter to match IPv6 Traffic Class field
# IPv6 header: offset 0, Traffic Class starts at bit 4

sudo tc filter add dev eth0 parent 1:0 \
  protocol ipv6 u32 \
  match ip6 protocol 17 0xff \
  match ip6 dport 5060 0xffff \
  action dsmark mask 0x3 value 0xb8  # EF = 0xb8 in TC byte
```

## Python Script for DSCP Policy Management

```python
#!/usr/bin/env python3
# apply_ipv6_dscp.py - Apply DSCP marking rules

import subprocess

DSCP_RULES = [
    # (protocol, port_type, port_range, dscp_class, description)
    ('udp', 'dport', '5060', 'CS5', 'SIP Signaling'),
    ('udp', 'dport', '10000:20000', '46', 'RTP Media (EF)'),
    ('tcp', 'dport', '22', 'AF31', 'SSH Interactive'),
    ('tcp', 'dport', '443', 'CS0', 'HTTPS Best Effort'),
    ('tcp', 'dport', '1935', 'CS3', 'RTMP Streaming'),
]

def apply_dscp_rule(proto, port_type, port, dscp, desc):
    """Apply ip6tables DSCP marking rule."""
    if dscp.startswith('CS') or dscp.startswith('AF') or dscp == 'EF':
        set_dscp = f'--set-dscp-class {dscp}'
    else:
        set_dscp = f'--set-dscp {dscp}'

    cmd = (
        f'ip6tables -t mangle -A PREROUTING '
        f'-p {proto} --{port_type} {port} '
        f'-j DSCP {set_dscp}'
    )
    result = subprocess.run(cmd.split(), capture_output=True)
    if result.returncode == 0:
        print(f"Applied DSCP {dscp} for {desc}")
    else:
        print(f"Error applying {desc}: {result.stderr.decode()}")

for rule in DSCP_RULES:
    apply_dscp_rule(*rule)
```

## Verifying DSCP Marking

```bash
# Capture and verify DSCP marks on IPv6 packets
sudo tcpdump -i eth0 -nn ip6 -v | grep "class 0x"

# Filter for specific DSCP values
# DSCP EF = 0x2e in Traffic Class field
sudo tcpdump -i eth0 -nn "ip6[1] & 0xfc == 0xb8"  # EF marking

# Use tshark for structured output
sudo tshark -i eth0 -f "ip6" \
  -T fields \
  -e ip6.src \
  -e ip6.dsfield.dscp \
  -e ip.proto | head -20

# Verify marking is preserved through router
# (Check at destination that DSCP was not remarked)
```

DSCP marking for IPv6 using ip6tables or nftables enables end-to-end QoS differentiation from the traffic source, with EF marking for VoIP RTP streams and CS values for signaling and network control being the most common and impactful marking policies to implement.
