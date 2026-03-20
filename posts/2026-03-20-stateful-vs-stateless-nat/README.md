# How to Understand Stateful vs Stateless NAT

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Networking, NAT, IPv4, Architecture

Description: Learn the difference between stateful and stateless NAT, when each is used, and how they compare in performance and use cases.

## Stateful NAT

Stateful NAT maintains a **connection tracking table** that records active sessions. For each NAT translation, the device stores:

- Source IP and port (original)
- Translated IP and port
- Destination IP and port
- Protocol and connection state (TCP state, UDP timeout)

### How Stateful NAT Works

```text
Client: 192.168.1.10:54321 → 8.8.8.8:53

NAT creates entry:
  192.168.1.10:54321 ↔ 203.0.113.1:1024 → 8.8.8.8:53

Reply arrives: 8.8.8.8:53 → 203.0.113.1:1024
NAT looks up table → translates back to 192.168.1.10:54321
```

### Examples of Stateful NAT

- Linux netfilter (iptables/nftables) with conntrack
- Cisco IOS NAT
- AWS NAT Gateway
- pfSense NAT
- Most home routers

### Advantages

- Works with PAT (many hosts, one public IP)
- Return traffic automatically matched to correct internal host
- Provides incidental firewall (stateful packet inspection)
- Handles TCP, UDP, ICMP with appropriate state

### Disadvantages

- Memory-intensive (state table per connection)
- Limits on concurrent sessions (conntrack max)
- Single point of failure (state is local)
- Complexity in HA/failover scenarios

## Stateless NAT

Stateless NAT performs **per-packet translation** without maintaining any session state. Each packet is translated independently based on configured rules.

### How Stateless NAT Works

```text
Rule: Map 203.0.113.x → 192.168.1.x (1:1 prefix mapping)
203.0.113.10 → 192.168.1.10  (always, no state needed)
203.0.113.20 → 192.168.1.20  (always, no state needed)
```

### Examples of Stateless NAT

- NAT66 (IPv6 prefix translation, RFC 6296)
- NAT-PT (deprecated)
- Some hardware NAT implementations
- Basic 1:1 NAT without port translation

### Advantages

- No memory overhead for state tables
- No session limits
- Easily distributable across multiple devices (no shared state)
- Deterministic, predictable behavior

### Disadvantages

- Cannot support PAT (requires state for port tracking)
- No protection against unsolicited inbound packets
- Requires dedicated IP per internal host
- Does not handle protocols that embed IP addresses in payloads

## Comparison Table

| Feature | Stateful NAT | Stateless NAT |
|---------|-------------|--------------|
| Session tracking | Yes | No |
| PAT support | Yes | No (1:1 only) |
| Memory overhead | High | Minimal |
| Session limits | Yes | No |
| Firewall effect | Yes | No |
| HA/failover | Complex | Simple |
| Performance | Moderate | High |
| Use cases | Typical home/enterprise | Network prefix translation, large-scale 1:1 |

## Stateless NAT in Practice: NAT66

RFC 6296 defines NPTv6 (Network Prefix Translation for IPv6) as a stateless 1:1 prefix mapping:

```text
Internal: fd00::/48
External: 2001:db8::/48

fd00::1 ↔ 2001:db8::1
fd00::2 ↔ 2001:db8::2
```

## Key Takeaways

- Stateful NAT tracks connections and is required for PAT (many-to-one).
- Stateless NAT is simpler, faster, and scales better but only works for 1:1 mappings.
- Most enterprise and home NAT uses stateful NAT with conntrack.
- Stateless NAT is used in high-performance scenarios (datacenter routers) and IPv6 prefix translation.

**Related Reading:**

- [How to Understand NAT and Its Impact on End-to-End Connectivity](https://oneuptime.com/blog/post/2026-03-20-nat-end-to-end-connectivity/view)
- [How to Scale NAT for Large Networks](https://oneuptime.com/blog/post/2026-03-20-scale-nat-large-networks/view)
- [How to Configure PAT (Port Address Translation)](https://oneuptime.com/blog/post/2026-03-20-configure-pat-nat-overload/view)
