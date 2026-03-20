# How to Analyze IPv6 Multicast Traffic in Wireshark

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Wireshark, IPv6, Multicast, MLD, ICMPv6, Packet Analysis

Description: A guide to analyzing IPv6 multicast traffic in Wireshark, including MLD messages, NDP multicast, and application multicast group membership.

IPv6 uses multicast extensively — it replaces IPv4 broadcast, powers Neighbor Discovery Protocol, and supports multicast applications. Analyzing multicast traffic in Wireshark is essential for diagnosing NDP failures, MLD issues, and multicast application problems.

## IPv6 Multicast Address Ranges

| Prefix | Scope | Common Uses |
|---|---|---|
| ff02::/16 | Link-local | NDP, MLD, routing protocols |
| ff05::/16 | Site-local | Site-wide services |
| ff0e::/16 | Global | Internet multicast |
| ff02::1 | All-nodes | Router Advertisements |
| ff02::2 | All-routers | Router Solicitations |
| ff02::16 | All MLDv2 routers | MLD reports |
| ff02::fb | mDNS | Bonjour/Avahi |
| ff02::1:2 | All DHCP relay agents | DHCPv6 |

## Display Filters for IPv6 Multicast

```wireshark
# Show ALL IPv6 multicast traffic
ipv6.dst == ff00::/8

# Show link-local multicast only (ff02::/16)
ipv6.dst == ff02::/16

# Show specific multicast groups
ipv6.dst == ff02::1    # All-nodes (Router Advertisements)
ipv6.dst == ff02::2    # All-routers (Router Solicitations)
ipv6.dst == ff02::16   # All MLDv2 routers
ipv6.dst == ff02::fb   # mDNS

# Show solicited-node multicast addresses (NDP)
ipv6.dst == ff02::1:ff00:0/104

# Show DHCPv6 multicast
ipv6.dst == ff02::1:2
```

## Multicast Listener Discovery (MLD) Analysis

```wireshark
# Show all MLD messages
mld

# Show MLD v1 messages (types 130-132)
icmpv6.type == 130   # Multicast Listener Query
icmpv6.type == 131   # MLDv1 Listener Report
icmpv6.type == 132   # Listener Done

# Show MLD v2 messages
icmpv6.type == 143   # MLDv2 Report

# Show MLD queries from the router
icmpv6.type == 130

# Show hosts reporting group membership
icmpv6.type == 131 || icmpv6.type == 143

# Show which groups a host has joined
icmpv6.type == 131 || icmpv6.type == 143
# Expand the ICMPv6 layer to see the multicast address
```

## Solicited-Node Multicast (NDP)

Every unicast IPv6 address automatically has a corresponding solicited-node multicast address (`ff02::1:ffXX:XXXX`). NDP Neighbor Solicitations are sent to this address:

```wireshark
# Show Neighbor Solicitations to solicited-node multicast
icmpv6.type == 135 && ipv6.dst == ff02::1:ff00:0/104

# Find NS for a specific address (2001:db8::10 → ff02::1:ff00:0010)
icmpv6.type == 135 && ipv6.dst == ff02::1:ff00:0010
```

## mDNS Multicast Traffic

```wireshark
# Show mDNS traffic (IPv6 multicast DNS)
ipv6.dst == ff02::fb && udp.port == 5353

# Show mDNS queries
dns.flags.response == 0 && ipv6.dst == ff02::fb

# Show mDNS responses
dns.flags.response == 1 && ipv6.dst == ff02::fb
```

## Count Multicast Groups in a Capture

```bash
# List all multicast destination addresses with packet counts
tshark -r capture.pcap \
  -Y "ipv6.dst == ff00::/8" \
  -T fields -e ipv6.dst | \
  sort | uniq -c | sort -rn | head -20
```

## Analyzing Router Advertisements via Multicast

```wireshark
# Show Router Advertisements (sent to ff02::1, all-nodes multicast)
icmpv6.type == 134

# Show Router Solicitations (sent to ff02::2, all-routers multicast)
icmpv6.type == 133
```

## Practical Multicast Debugging

### Problem: Host Not Getting SLAAC Address

```wireshark
# Check if Router Advertisements are being sent
icmpv6.type == 134

# Check if Router Solicitations are being sent
icmpv6.type == 133

# If RS exists but no RA: router is not responding
# If neither RS nor RA: check if ICMPv6 multicast is blocked
```

### Problem: mDNS Discovery Not Working

```wireshark
# Check if mDNS queries are being sent
dns.flags.response == 0 && ipv6.dst == ff02::fb

# Check if mDNS responses are being received
dns.flags.response == 1 && (ipv6.src == fe80::/10)
```

IPv6 multicast analysis in Wireshark uncovers issues that are invisible at the unicast layer — from missing Router Advertisements that prevent SLAAC addressing to MLD failures that break multicast application delivery.
