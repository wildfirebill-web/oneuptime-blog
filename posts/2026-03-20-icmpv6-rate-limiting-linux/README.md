# How to Configure ICMPv6 Rate Limiting on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ICMPv6, Rate Limiting, Linux, Sysctl, Kernel Configuration

Description: Configure Linux kernel ICMPv6 rate limiting using sysctl, understand the ratelimit and ratemask parameters, and tune them for different environments.

## Introduction

Linux automatically rate-limits ICMPv6 error message generation to prevent a router or host from flooding the network with error messages. The rate limiting is controlled by two sysctl parameters: `net.ipv6.icmp.ratelimit` (the interval between messages in milliseconds) and `net.ipv6.icmp.ratemask` (a bitmask that controls which message types are rate-limited). Understanding these parameters allows tuning for both DoS protection and high-performance environments.

## Linux ICMPv6 Rate Limiting Parameters

```bash
# View current rate limiting settings

sysctl net.ipv6.icmp.ratelimit
# Default: 1000 (1000ms = 1 error per second per source)

sysctl net.ipv6.icmp.ratemask
# Default: 0000001110000000 in binary representation
# = bitmask of ICMPv6 types that are rate-limited

# Interpret the ratemask as a bitmask:
# bit N = 1 means ICMPv6 type N is rate-limited
python3 -c "
mask = $(cat /proc/sys/net/ipv6/icmp/ratemask)
print(f'Ratemask: {mask} ({mask:016b} binary)')
for t in range(16):
    if mask & (1 << t):
        print(f'  Type {t} is rate-limited')
"
```

## Tuning ratelimit

```bash
# View current setting
cat /proc/sys/net/ipv6/icmp/ratelimit
# 1000 = 1 error per second (default)

# High-traffic server: allow up to 100 errors per second
sudo sysctl -w net.ipv6.icmp.ratelimit=10

# Conservative setting: 1 error per 10 seconds
sudo sysctl -w net.ipv6.icmp.ratelimit=10000

# Effectively disable rate limiting (use only if you have other protections)
sudo sysctl -w net.ipv6.icmp.ratelimit=0

# Make persistent in /etc/sysctl.conf or /etc/sysctl.d/
echo "net.ipv6.icmp.ratelimit=100" | sudo tee /etc/sysctl.d/ipv6-icmp.conf
sudo sysctl -p /etc/sysctl.d/ipv6-icmp.conf
```

## Tuning ratemask

```bash
# The ratemask determines WHICH ICMPv6 types are rate-limited
# Default includes Types 1, 2, 3 (error messages)

# Calculate a custom ratemask
python3 << 'EOF'
# Types to rate-limit: 1 (Dest Unreachable), 3 (Time Exceeded), 4 (Parameter Problem)
# Type 2 (Packet Too Big) should NOT be rate-limited (would break PMTUD)
rate_limit_types = [1, 3, 4]   # Exclude Type 2!
mask = 0
for t in rate_limit_types:
    mask |= (1 << t)
print(f"Ratemask value: {mask}")
print(f"Binary: {mask:016b}")
EOF

# Apply the custom ratemask (replace <VALUE> with the calculated number)
# Example: rate limit types 1, 3, 4 but NOT 2 (Packet Too Big)
sudo sysctl -w net.ipv6.icmp.ratemask=26  # 11010 binary = types 1, 3, 4

# Verify the types being rate-limited
python3 -c "
mask = 26
for t in range(16):
    status = 'rate-limited' if mask & (1 << t) else 'NOT rate-limited'
    if t in [1,2,3,4]:
        print(f'Type {t}: {status}')
"
```

## Rate Limiting Specific Types with ip6tables

For more granular control than the kernel sysctl:

```bash
# Rate limit specific ICMPv6 types using ip6tables
# This applies to incoming ICMPv6 from the network

# Allow Packet Too Big without rate limiting (critical for PMTUD)
sudo ip6tables -I INPUT 1 -p icmpv6 --icmpv6-type packet-too-big -j ACCEPT

# Rate limit Echo Request (ping flood protection)
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type echo-request \
    -m limit --limit 20/second --limit-burst 40 -j ACCEPT
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type echo-request -j DROP

# Rate limit Neighbor Solicitation
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type neighbor-solicitation \
    -m limit --limit 50/second --limit-burst 100 -j ACCEPT
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type neighbor-solicitation -j DROP

# Global rate limit for all other ICMPv6
sudo ip6tables -A INPUT -p icmpv6 \
    -m limit --limit 100/second --limit-burst 200 -j ACCEPT
sudo ip6tables -A INPUT -p icmpv6 -j DROP
```

## Monitoring Rate Limiting Effects

```bash
# Check if rate limiting is dropping ICMPv6
cat /proc/net/snmp6 | grep -E "Icmp6|OutErrors"

# Compare with tcpdump to see actual traffic rate
sudo tcpdump -i eth0 -q "icmp6" 2>&1 | \
    awk '{count++} NR%10==0 {print count " ICMPv6 packets in last 10"}'

# Check ip6tables hit counts on rate-limit rules
sudo ip6tables -L INPUT -v -n | grep "icmpv6.*limit"

# Monitor neighbor table for exhaustion (related to NS rate limiting)
watch -n 1 'ip -6 neigh show | wc -l'
# If this grows rapidly: NS flood or neighbor table exhaustion
```

## Recommended Settings by Environment

```python
RATE_LIMIT_PROFILES = {
    "workstation": {
        "ratelimit":  1000,  # 1/second (default)
        "description": "Conservative default; adequate for desktop use"
    },
    "server": {
        "ratelimit":  100,   # 10/second
        "description": "More responsive for servers handling many connections"
    },
    "router": {
        "ratelimit":  50,    # 20/second
        "description": "Higher throughput; router handles many flows"
    },
    "ddos_protection": {
        "ratelimit":  5000,  # 0.2/second
        "description": "Aggressive limiting when under attack"
    },
}

for profile, config in RATE_LIMIT_PROFILES.items():
    print(f"{profile}: ratelimit={config['ratelimit']}ms - {config['description']}")
```

## Conclusion

Linux ICMPv6 rate limiting protects against error storms and DoS attacks using the `net.ipv6.icmp.ratelimit` sysctl. The `ratemask` controls which types are rate-limited - never include Type 2 (Packet Too Big) in the ratemask, as rate-limiting it breaks PMTUD. The default settings (1 error per second, types 1-4 rate-limited) are appropriate for most environments. For high-performance servers or routers, reduce the ratelimit interval (to 100ms or less). For DoS protection, combine kernel rate limiting with ip6tables `--limit` rules for per-type and per-source granularity.
