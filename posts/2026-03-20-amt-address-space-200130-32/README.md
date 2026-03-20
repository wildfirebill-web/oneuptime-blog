# How to Understand the AMT Address Space (2001:3::/32)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, AMT, Automatic Multicast Tunneling, RFC 7450, Multicast, Networking

Description: Understand the AMT (Automatic Multicast Tunneling) address space 2001:3::/32 used to tunnel IPv6 multicast traffic over IPv4 unicast networks.

## Introduction

Automatic Multicast Tunneling (AMT), defined in RFC 7450, uses the `2001:3::/32` address block to provide IPv6 multicast connectivity to hosts that lack native multicast routing. AMT encapsulates multicast traffic in UDP and forwards it over unicast-only networks.

## How AMT Works

```mermaid
graph LR
    Client["AMT Pseudo-IF\n(receives 2001:3:: addr)"]
    AMT_GW["AMT Gateway\n(last-hop multicast router)"]
    Internet["Internet\n(unicast only)"]
    AMT_Relay["AMT Relay\n(first-hop multicast router)"]
    Source["Multicast Source"]

    Client --> AMT_GW
    AMT_GW <-->|"UDP tunnel"| AMT_Relay
    AMT_Relay <--> Source
    Source --> Internet
```

## AMT Address Format

```
2001:3::/32 — AMT prefix
  Allocated by IANA for AMT use
  Not for general routing

AMT pseudo-interface address structure:
  2001:3:: + <AMT relay IPv4 address> (32 bits)

Example: 2001:3::c000:0201 (relay at 192.0.2.1)
```

## Checking for AMT Addresses

```python
import ipaddress

def is_amt_address(addr: str) -> bool:
    """Check if an IPv6 address is in the AMT block."""
    try:
        a = ipaddress.IPv6Address(addr)
        return a in ipaddress.IPv6Network("2001:3::/32")
    except ValueError:
        return False

print(is_amt_address("2001:3::1"))       # True
print(is_amt_address("2001:3::c000:1"))  # True (relay at 192.0.0.1)
```

## Linux AMT Tunnel Setup

```bash
# Install amtrelayd (AMT relay daemon)
sudo apt-get install amtrelayd

# Configure AMT relay
# /etc/amtrelayd.conf
cat > /etc/amtrelayd.conf << 'EOF'
# AMT relay configuration
anycast-address 2001:3::1
relay-address 203.0.113.1
EOF

# Start the relay
sudo systemctl start amtrelayd

# Verify AMT tunnel interface
ip -6 addr show | grep "2001:3::"
```

## Filtering AMT Addresses

```bash
# AMT addresses should not be routed externally by default
ip6tables -A FORWARD -s 2001:3::/32 -j LOG --log-prefix "AMT: "
ip6tables -A FORWARD -d 2001:3::/32 \
  -i eth-external -j DROP
```

## Conclusion

The `2001:3::/32` AMT address space enables IPv6 multicast over unicast networks. It's primarily used in enterprise multicast deployments and IPTV scenarios. Filter AMT addresses at network boundaries and monitor AMT relay availability with OneUptime for multicast service health.
