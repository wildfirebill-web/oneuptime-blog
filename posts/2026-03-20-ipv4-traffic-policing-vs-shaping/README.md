# How to Configure IPv4 Traffic Policing vs Shaping and When to Use Each

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Traffic Control, Policing, Shaping, tc, IPv4, QoS, Linux

Description: Understand the difference between IPv4 traffic policing (drop excess) and shaping (delay excess), configure both with Linux tc, and choose the right approach for your use case.

## Introduction

Traffic policing and traffic shaping are two different approaches to enforcing bandwidth limits on IPv4 traffic:

- **Policing**: Drops or re-marks packets that exceed the rate limit immediately. No buffering.
- **Shaping**: Delays excess packets in a queue until they can be sent within the rate limit. Smooths out bursts.

## Key Differences

| Aspect | Policing | Shaping |
|--------|---------|---------|
| Excess packets | Dropped | Queued/delayed |
| Latency impact | None (drops immediately) | Adds buffering latency |
| Memory usage | Minimal | Queue buffer required |
| Application side effect | Retransmissions | Smooth but delayed delivery |
| Best for | Ingress limiting, multi-tenant | Egress smoothing, WAN links |

## Configuring Traffic Shaping (TBF)

The Token Bucket Filter (TBF) qdisc smooths bursts by queuing excess packets:

```bash
# Shape outbound traffic on eth0 to 10 Mbit/s
# TBF allows a burst of 32kb before queuing begins
sudo tc qdisc add dev eth0 root handle 1: tbf \
  rate 10mbit \         # Sustained rate
  burst 32kbit \        # Burst bucket size
  latency 200ms         # Maximum time a packet can wait in the queue

# Verify
sudo tc -s qdisc show dev eth0
```

**When to use shaping**: On WAN uplinks where you want to smooth traffic and avoid congestion at the ISP — preventing router buffer bloat. Also useful for backing off a bulk transfer without causing retransmissions.

## Configuring Traffic Policing

Policing drops packets immediately when the rate is exceeded. It is most commonly used on ingress with an IFB device or directly on ingress qdiscs:

```bash
# Police ingress traffic to 10 Mbit/s (drop exceeding packets)
sudo tc qdisc add dev eth0 ingress

sudo tc filter add dev eth0 parent ffff: protocol ip u32 \
  match u32 0 0 \
  police rate 10mbit \    # Rate limit
  burst 200k \            # Burst allowance
  conform-exceed drop \   # Action on exceeding traffic: drop
  flowid :1
```

**When to use policing**: For inbound traffic limiting (ingress), where you cannot buffer because you haven't received the packets yet. Also appropriate for multi-tenant environments where you want hard enforcement.

## HTB with Shaping (Production Pattern)

For more granular shaping with multiple traffic classes:

```bash
# Root HTB qdisc with a default class
sudo tc qdisc add dev eth0 root handle 1: htb default 30

# High-priority class for interactive traffic (SSH, DNS)
sudo tc class add dev eth0 parent 1: classid 1:10 htb \
  rate 2mbit \
  ceil 10mbit

# Low-priority class for bulk transfers
sudo tc class add dev eth0 parent 1: classid 1:30 htb \
  rate 1mbit \
  ceil 5mbit

# Classify SSH to the high-priority class
sudo tc filter add dev eth0 protocol ip parent 1: prio 1 u32 \
  match ip dport 22 0xffff \
  flowid 1:10
```

## Decision Guide

**Use policing when:**
- You need to enforce hard limits on ingress traffic
- You're managing bandwidth for untrusted tenants or clients
- Simplicity is more important than smooth delivery
- The protocol handles retransmissions well (TCP)

**Use shaping when:**
- You need smooth, bursty egress traffic (video streaming, file uploads)
- You want to prevent congestion on the outbound link
- Avoiding TCP retransmissions is a priority
- You have predictable, controlled traffic sources

## Removing Rules

```bash
# Remove all tc rules
sudo tc qdisc del dev eth0 root
sudo tc qdisc del dev eth0 ingress
```

## Conclusion

Choose shaping for egress smoothing and policing for ingress enforcement. In practice, use shaping on your outbound WAN interface with HTB for multi-class prioritization, and policing on ingress to enforce hard per-tenant limits.
