# How to Configure Path MTU Discovery for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Path MTU Discovery, PMTUD, MTU, Networking

Description: Configure Path MTU Discovery for IPv6 on Linux and other systems, understand how PMTU caching works, and ensure PMTUD operates correctly in your environment.

## Introduction

Path MTU Discovery (PMTUD) for IPv6 is defined in RFC 8201 and is essential for efficient packet delivery. Unlike IPv4 where router fragmentation can silently handle oversized packets, IPv6 requires PMTUD to avoid packet loss. When a packet is too large for a link, the router sends an ICMPv6 Packet Too Big message and drops the packet — PMTUD allows the source to learn this and adjust.

## How IPv6 PMTUD Works

```
PMTUD Process:

1. Source sends packet sized to local interface MTU (e.g., 1500 bytes)
2. Router with smaller-MTU next link:
   - Drops the packet
   - Sends ICMPv6 Packet Too Big (Type 2) with the next-link MTU
3. Source receives PTB message:
   - Updates PMTU cache for this destination
   - Resends packet at the reduced size
4. Process repeats if further bottlenecks exist
5. PMTU cache entries expire (default: 10 minutes on Linux)
   - Source retries with full MTU to detect path changes
```

## Configuring PMTUD on Linux

PMTUD is enabled by default on Linux but can be tuned:

```bash
# Check PMTU discovery status per interface
cat /proc/sys/net/ipv6/conf/eth0/path_mtu_discovery
# 1 = enabled (default)

# Enable PMTU discovery globally
sudo sysctl -w net.ipv6.conf.all.path_mtu_discovery=1

# Make persistent
echo "net.ipv6.conf.all.path_mtu_discovery=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# View current PMTU cache (cached path MTU values per destination)
ip -6 route show cache

# View PMTU cache with more detail
ip -6 route show cache | grep mtu

# Force PMTU rediscovery by flushing the cache
sudo ip -6 route flush cache

# View PMTU statistics
cat /proc/net/snmp6 | grep -i pmtu
# Key counters:
# Ip6InTooBigErrors   - received Packet Too Big messages
# Ip6FragCreates      - fragments created by this host
```

## Checking PMTU for a Specific Destination

```bash
# Check if a PMTU cache entry exists for a destination
ip -6 route get 2001:db8::1

# Example output showing PMTU:
# 2001:db8::1 from :: via fe80::1 dev eth0 src 2001:db8::100
#    cache  expires 594sec mtu 1280

# Test PMTUD interactively using ping6 with large packet sizes
ping6 -M do -s 1452 2001:db8::1
# -M do: "prohibit fragmentation, even local" (equivalent to DF bit in IPv4)
# -s 1452: data size (1452 + 8 ICMPv6 + 40 IPv6 = 1500 bytes total)

# If path MTU is smaller, you should see:
# "Message too long: mtu=1280"
```

## Firewall Configuration for PMTUD

PMTUD fails silently when firewalls block ICMPv6 Packet Too Big. This is the most common cause of "black hole" connectivity issues:

```bash
# Allow ICMPv6 Packet Too Big through the firewall (CRITICAL)
# Using ip6tables
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type packet-too-big -j ACCEPT
sudo ip6tables -A OUTPUT -p icmpv6 --icmpv6-type packet-too-big -j ACCEPT
sudo ip6tables -A FORWARD -p icmpv6 --icmpv6-type packet-too-big -j ACCEPT

# Using nftables
# nft add rule ip6 filter input icmpv6 type packet-too-big accept
# nft add rule ip6 filter forward icmpv6 type packet-too-big accept

# Verify existing rules allow PTB (look for packet-too-big or type 2)
sudo ip6tables -L -v | grep -i "packet-too-big\|icmpv6"

# Test if PMTU messages are reaching the source
sudo tcpdump -i eth0 "icmp6 and ip6[40] == 2"
```

## PMTU Black Hole Detection

Linux has built-in black hole detection to handle cases where PTB messages are silently dropped:

```bash
# Check black hole detection settings
cat /proc/sys/net/ipv6/route/mtu_expires
# Default: 600 seconds (10 minutes)

# PMTU cache age - after this, source retries with original MTU
# to detect path changes

# Check if TCP is using PMTUD
cat /proc/sys/net/ipv4/tcp_mtu_probing
# 0 = disabled
# 1 = enabled when black hole detected
# 2 = always enabled (uses MSS_BASE)

# Enable TCP MTU probing (helps when PMTUD fails)
sudo sysctl -w net.ipv4.tcp_mtu_probing=1
```

## Verifying PMTUD End-to-End

```python
import subprocess
import re

def check_pmtu_to_destination(destination: str) -> dict:
    """
    Check the cached PMTU to a destination using 'ip route get'.
    Returns PMTU info or None if no cached entry.
    """
    result = subprocess.run(
        ["ip", "-6", "route", "get", destination],
        capture_output=True, text=True
    )
    output = result.stdout

    mtu_match = re.search(r'mtu (\d+)', output)
    expires_match = re.search(r'expires (\d+)sec', output)
    via_match = re.search(r'via (\S+)', output)

    return {
        "destination": destination,
        "cached_mtu": int(mtu_match.group(1)) if mtu_match else None,
        "expires_in": int(expires_match.group(1)) if expires_match else None,
        "via": via_match.group(1) if via_match else None,
        "raw": output.strip(),
    }

result = check_pmtu_to_destination("2001:db8::1")
if result["cached_mtu"]:
    print(f"PMTU to {result['destination']}: {result['cached_mtu']} bytes")
    print(f"Cache expires in: {result['expires_in']} seconds")
else:
    print(f"No cached PMTU for {result['destination']} (will use interface MTU)")
```

## Conclusion

IPv6 PMTUD is enabled by default on modern systems and requires no manual configuration in typical environments. The critical operational requirement is ensuring ICMPv6 Packet Too Big messages are never blocked by firewalls. PMTU cache entries are time-limited, so the system naturally adapts to path changes. When connectivity issues occur with large transfers but small packets work fine, always check for PMTUD black holes by verifying ICMPv6 type 2 messages can reach the source.
