# How to Configure ICMP Unreachable Rate Limiting

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ICMP, Linux, Rate Limiting, Security, Networking, Kernel

Description: Configure Linux kernel and iptables ICMP unreachable rate limiting to prevent your server from being used as an ICMP amplifier while maintaining legitimate error reporting.

## Introduction

When a Linux host generates ICMP Unreachable messages (e.g., for every UDP packet to a closed port), an attacker can exploit this to amplify traffic or exhaust CPU. The Linux kernel has built-in ICMP rate limiting for outbound error messages, and iptables provides additional control. Configuring these protects your server without breaking legitimate network diagnostics.

## Linux Kernel ICMP Rate Limiting

The kernel controls how often it sends ICMP error messages using two sysctl parameters:

```bash
# View current rate limit settings
sysctl net.ipv4.icmp_ratelimit
# Default: 1000 (allows burst of this many tokens, refills at 100/sec)

sysctl net.ipv4.icmp_ratemask
# Default: 6168 = bitmask of ICMP types subject to rate limiting
# 6168 = bits 3,4,5,8,9,11,12 (dest unreachable, redirect, time exceeded, etc.)

# Set rate limit: 100ms minimum between ICMP error messages
sysctl -w net.ipv4.icmp_ratelimit=100

# Persist settings
echo "net.ipv4.icmp_ratelimit=100" >> /etc/sysctl.conf
sysctl -p
```

## Understanding icmp_ratemask

```bash
# The bitmask controls WHICH types are rate limited
# Bit position = ICMP type number
# Default 6168 = 0001100000011000 in binary
# Active bits: 3,4,5,11,12 (destination unreachable types + time exceeded)

# To add type 8 (echo request) to the rate-limited types:
# 6168 + bit 8 (256) = 6424
sysctl -w net.ipv4.icmp_ratemask=6424

# To rate-limit ALL ICMP error types (recommended for high-traffic servers)
sysctl -w net.ipv4.icmp_ratemask=65535
```

## iptables Rate Limiting for Outbound Unreachables

```bash
# Limit outbound ICMP Port Unreachable messages (type 3 code 3)
# This protects against being used as an ICMP amplifier
iptables -A OUTPUT -p icmp --icmp-type port-unreachable \
  -m limit --limit 10/sec --limit-burst 20 \
  -j ACCEPT

iptables -A OUTPUT -p icmp --icmp-type port-unreachable -j DROP

# Limit all outbound ICMP unreachable messages
iptables -A OUTPUT -p icmp --icmp-type destination-unreachable \
  -m limit --limit 20/sec --limit-burst 50 \
  -j ACCEPT
iptables -A OUTPUT -p icmp --icmp-type destination-unreachable -j DROP
```

## Per-Source Rate Limiting for Unreachables

```bash
# Prevent a single source from triggering mass ICMP unreachables
# (e.g., scanner hitting many closed UDP ports)
iptables -A INPUT -p udp \
  -m hashlimit \
  --hashlimit-name udp-protection \
  --hashlimit-mode srcip \
  --hashlimit-upto 100/sec \
  --hashlimit-burst 200 \
  -j ACCEPT

# Excess UDP from same source is dropped silently (no ICMP generated)
iptables -A INPUT -p udp -j DROP
```

## Monitoring ICMP Rate Limiting in Action

```bash
# Check if kernel rate limiting is active
nstat -a | grep IcmpRateLimit
# InCsumErrors, RateLimitedOut, etc.

# More detailed kernel ICMP stats
cat /proc/net/snmp | awk '/^Icmp:/{getline; print}'

# Watch for OutDestUnreachs and OutTimeExcds counters
watch -n 1 "nstat -z | grep -E 'IcmpOut|Icmp.*Unreachable'"
```

## Conclusion

ICMP unreachable rate limiting is a defensive measure that prevents your server from being an amplifier for UDP-based scanning attacks while preserving legitimate error reporting. The Linux kernel's built-in icmp_ratelimit handles the baseline, while iptables provides fine-grained per-type and per-source controls. Set rates generous enough for monitoring but restrictive enough to prevent abuse.
