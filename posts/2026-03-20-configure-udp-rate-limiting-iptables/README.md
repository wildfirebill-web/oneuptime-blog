# How to Configure UDP Rate Limiting with iptables

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: UDP, iptables, Rate Limiting, Linux, Networking, Security, Firewall

Description: Configure iptables rules to rate limit UDP traffic per source IP, per port, and globally to prevent abuse, protect services, and control UDP bandwidth usage.

## Introduction

UDP rate limiting with iptables protects services from UDP floods, prevents a single client from monopolizing bandwidth, and implements per-service traffic policies. The `limit` and `hashlimit` modules provide flexible rate limiting: `limit` applies a global rate cap, while `hashlimit` limits per source IP, destination, or source-destination pair.

## Global UDP Rate Limiting

```bash
# Limit all UDP to 1000 packets/second globally:
iptables -A INPUT -p udp -m limit \
  --limit 1000/sec \
  --limit-burst 2000 \
  -j ACCEPT

iptables -A INPUT -p udp -j DROP
# Packets beyond 1000/sec (after burst of 2000) are dropped

# Limit UDP on a specific port:
iptables -A INPUT -p udp --dport 5000 \
  -m limit --limit 500/sec --limit-burst 1000 \
  -j ACCEPT
iptables -A INPUT -p udp --dport 5000 -j DROP
```

## Per-Source IP Rate Limiting (hashlimit)

```bash
# Limit each source IP to 100 UDP packets/second on port 5000:
iptables -A INPUT -p udp --dport 5000 \
  -m hashlimit \
  --hashlimit-name udp_5000 \
  --hashlimit-above 100/sec \
  --hashlimit-burst 200 \
  --hashlimit-mode srcip \
  -j DROP

# Explanation:
# --hashlimit-name: name for the hash table (unique per rule)
# --hashlimit-above 100/sec: drop packets ABOVE this rate
# --hashlimit-burst 200: allow initial burst of 200 packets
# --hashlimit-mode srcip: rate limit per source IP address

# More aggressive: limit DNS to 50 queries/second per source IP:
iptables -A INPUT -p udp --dport 53 \
  -m hashlimit \
  --hashlimit-name dns_limit \
  --hashlimit-above 50/sec \
  --hashlimit-burst 100 \
  --hashlimit-mode srcip \
  -j DROP
```

## Rate Limiting by Source IP and Port

```bash
# Rate limit per src IP and src port combination:
# Useful for protocols where the same source port indicates the same flow
iptables -A INPUT -p udp \
  -m hashlimit \
  --hashlimit-name udp_flow \
  --hashlimit-above 1000/sec \
  --hashlimit-burst 2000 \
  --hashlimit-mode srcip,srcport \
  -j DROP

# Per destination IP (protect specific servers differently):
iptables -A FORWARD -p udp -d 10.20.0.10 \
  -m hashlimit \
  --hashlimit-name udp_server10 \
  --hashlimit-above 5000/sec \
  --hashlimit-burst 10000 \
  --hashlimit-mode dstip \
  -j DROP
```

## Rate Limiting with LOG and DROP

```bash
# Log exceeding packets before dropping (useful for monitoring):
iptables -A INPUT -p udp --dport 5000 \
  -m hashlimit \
  --hashlimit-name udp_log \
  --hashlimit-above 200/sec \
  --hashlimit-burst 400 \
  --hashlimit-mode srcip \
  -m limit --limit 5/min \
  -j LOG --log-prefix "UDP rate limit: " --log-level 4

iptables -A INPUT -p udp --dport 5000 \
  -m hashlimit \
  --hashlimit-name udp_drop \
  --hashlimit-above 200/sec \
  --hashlimit-burst 400 \
  --hashlimit-mode srcip \
  -j DROP
```

## Apply Rate Limiting at Kernel Level (tc)

```bash
# iptables rate limits by packet count; tc limits by bandwidth

# Limit UDP to 50 Mbps on eth0:
tc qdisc add dev eth0 root handle 1: htb default 10
tc class add dev eth0 parent 1: classid 1:10 htb rate 50mbit ceil 50mbit

# Limit only UDP (using filter):
tc qdisc add dev eth0 root handle 1: htb default 20
tc class add dev eth0 parent 1: classid 1:10 htb rate 10mbit  # UDP class
tc class add dev eth0 parent 1: classid 1:20 htb rate 90mbit  # Everything else
tc filter add dev eth0 parent 1: protocol ip u32 \
  match ip protocol 17 0xff flowid 1:10  # Protocol 17 = UDP
```

## Verify and Monitor Rules

```bash
# Show rules with packet/byte counters:
iptables -L INPUT -n -v | grep -E "udp|DROP"

# Watch hashlimit table:
# Hashlimit state is in /proc/net/ipt_hashlimit/
cat /proc/net/ipt_hashlimit/udp_5000

# Reset counters:
iptables -Z INPUT

# Save rules permanently:
iptables-save > /etc/iptables/rules.v4
```

## Conclusion

iptables `hashlimit` is the right tool for per-source UDP rate limiting. Use `--hashlimit-mode srcip` for per-client limits, `--hashlimit-above` to define the threshold, and include a burst allowance for legitimate traffic spikes. For bandwidth-based limiting rather than packet count limiting, use `tc` with an HTB qdisc. Always test rate limits with normal traffic first to ensure legitimate users aren't affected, and monitor the hashlimit state table to understand actual usage patterns.
