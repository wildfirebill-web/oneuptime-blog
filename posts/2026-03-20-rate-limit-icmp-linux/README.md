# How to Rate-Limit ICMP Traffic on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ICMP, iptables, Linux, Rate Limiting, Security, Networking

Description: Configure iptables and nftables rate-limiting rules to protect your Linux server from ICMP floods while preserving legitimate monitoring and diagnostic traffic.

## Introduction

ICMP rate limiting prevents your server from being overwhelmed by ping floods while still allowing normal monitoring and diagnostic operations. Linux's iptables hashlimit and limit modules provide per-source and global rate limiting respectively. Both approaches let you specify a maximum packet rate with a burst allowance.

## Global Rate Limiting with iptables limit Module

The simplest approach: limit total ICMP echo requests to a safe rate:

```bash
# Allow ICMP echo at 5/second with burst of 10
# Packets exceeding this rate are dropped
iptables -A INPUT -p icmp --icmp-type echo-request \
  -m limit --limit 5/sec --limit-burst 10 \
  -j ACCEPT

# Drop ICMP echo requests that exceed the rate
iptables -A INPUT -p icmp --icmp-type echo-request -j DROP

# Verify: fast ping should show drops after the burst
ping -f 192.168.1.10   # flood ping from another host
```

## Per-Source Rate Limiting with hashlimit

The hashlimit module tracks rates per source IP, preventing one attacker from consuming the entire global allowance:

```bash
# Allow 2 ICMP echo requests per second PER SOURCE IP
# Each source gets an independent token bucket
iptables -A INPUT -p icmp --icmp-type echo-request \
  -m hashlimit \
  --hashlimit-name icmp-limit \
  --hashlimit-mode srcip \
  --hashlimit-upto 2/sec \
  --hashlimit-burst 5 \
  --hashlimit-htable-expire 30000 \
  -j ACCEPT

# Drop if over the per-source limit
iptables -A INPUT -p icmp --icmp-type echo-request -j DROP
```

## Rate Limiting with nftables

```bash
# /etc/nftables.conf
table inet filter {
  chain input {
    type filter hook input priority 0; policy drop;

    # Rate-limit ICMP echo requests: max 10/second globally
    icmp type echo-request \
      limit rate 10/second burst 20 packets \
      accept

    # Per-source rate limiting with meters (nftables 0.9.3+)
    icmp type echo-request \
      meter icmp-meter { ip saddr limit rate 5/second } \
      accept

    # Allow essential ICMP types without rate limiting
    icmp type { echo-reply, destination-unreachable, time-exceeded } accept

    # Drop excess ICMP
    icmp type echo-request drop
  }
}
```

## Rate Limiting ICMP at the Kernel Level

```bash
# Linux kernel ICMP rate limiting (affects outbound ICMP error messages)
# Rate at which the kernel sends ICMP unreachable/time-exceeded responses
sysctl net.ipv4.icmp_ratelimit
# Default: 1000 (100ms between ICMP error bursts)

# Set to 100ms minimum between ICMP error messages
sysctl -w net.ipv4.icmp_ratelimit=100

# Configure which ICMP types are rate-limited
sysctl net.ipv4.icmp_ratemask
# Default: 6168 — bitmask of ICMP types to rate-limit
# Bit 3 = Dest Unreachable, Bit 11 = TTL Exceeded
```

## Monitoring Rate Limit Effectiveness

```bash
# Watch iptables rule counters — drops indicate rate limiting is working
watch -n 2 "iptables -L INPUT -n -v | grep icmp"

# Check kernel ICMP statistics for rate-limited messages
cat /proc/net/snmp | grep Icmp
# InEchos: total echo requests received
# OutEchoReps: echo replies sent (lower than InEchos = limited)

# Also check: nstat -a | grep -i icmp
nstat -z | grep -i "icmp"
```

## Conclusion

ICMP rate limiting is a lightweight but effective protection against ping floods. Use the `limit` module for global rate control and `hashlimit` for per-source limits that prevent a single attacker from exhausting the global allowance. Always apply rate limiting separately from blocking — rate-limited ICMP provides operational value while protecting resources.
