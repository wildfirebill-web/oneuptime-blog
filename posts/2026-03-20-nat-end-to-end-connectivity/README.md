# How to Understand NAT and Its Impact on End-to-End Connectivity

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Networking, NAT, IPv4, Architecture

Description: Learn how NAT violates the end-to-end principle of the internet, what capabilities it breaks, and the trade-offs of using NAT vs IPv6.

## The End-to-End Principle

The original internet design assumed **end-to-end connectivity**: any host with an IP address can communicate directly with any other host. This enabled:

- Any host to act as a server
- Peer-to-peer applications
- Novel protocols without needing middleboxes to understand them
- Transparent address assignment

NAT fundamentally breaks this model.

## How NAT Breaks End-to-End

```text
IPv4 Internet Model (with NAT):

[192.168.1.10] → [NAT Router] → [Internet] → [External Host]
     ↑
  Private IP (not globally routable)
  Cannot be reached directly from outside
```

### What NAT Breaks

1. **Inbound connections** - external hosts cannot initiate connections to private IPs
2. **Self-hosting** - requires explicit port forwarding configuration
3. **Peer-to-peer** - requires NAT traversal (STUN/TURN/ICE)
4. **Protocol transparency** - some protocols (FTP, SIP, H.323) embed IPs in payloads
5. **IP-based security** - certificates and security models based on IP identity are complicated
6. **Multicast across NAT** - not supported in standard NAT

## Applications That Require Special NAT Handling

| Application | Problem | Solution |
|-------------|---------|----------|
| FTP (active mode) | Server connects back to client | Use passive FTP or FTP ALG |
| SIP/VoIP | Private IPs in SDP | STUN/TURN/SBC |
| BitTorrent | Peers can't reach you | Port forwarding or UPnP |
| Online gaming | Direct P2P fails | UPnP, hole punching |
| SSH server at home | Not reachable | Port forward or VPN |
| H.323 video | Complex NAT traversal | H.323 proxy/ALG |

## NAT as an Unintended Firewall

NAT provides **stateful packet filtering** as a side effect:

- Unsolicited inbound packets have no conntrack entry → silently dropped
- This blocks many attack vectors (port scans, unsolicited connections)
- However, this is security through NAT, not intentional firewall policy

## NAT vs IPv6

IPv6 was designed to restore end-to-end connectivity:

```text
IPv6 Design:
- 128-bit addresses: enough for every device to have a unique global address
- No NAT required by design
- Direct end-to-end communication restored
- Firewall functionality explicitly separated from addressing
```

Comparison:

| Feature | IPv4 + NAT | IPv6 |
|---------|-----------|------|
| Address space | Limited (shared) | Abundant |
| End-to-end | Broken | Restored |
| Inbound connections | Requires port forward | Works directly (with firewall) |
| P2P | Complex (NAT traversal) | Simple |
| Self-hosting | Requires configuration | Works out of box |

## When NAT Is Still Useful

Even with IPv6, NAT has legitimate uses:

1. **IPv4 address conservation** - unavoidable given IPv4 exhaustion
2. **Network renumbering** - change ISP without renumbering all internal hosts
3. **Privacy** - hide internal host count from external observers
4. **Load balancing** - multiple backends behind one IP (DNAT)
5. **Policy routing** - select source IP for different traffic types

## Key Takeaways

- NAT violates the end-to-end principle by making private hosts unreachable from outside.
- This breaks P2P applications, self-hosting, and transparent protocols (FTP, SIP).
- NAT's stateful filtering provides incidental security, but is not a substitute for a proper firewall.
- IPv6 was designed to restore end-to-end connectivity and eliminate NAT's necessity.

**Related Reading:**

- [How to Understand NAT Types (Full Cone, Restricted, Symmetric)](https://oneuptime.com/blog/post/2026-03-20-nat-types-full-cone/view)
- [How to Understand NAT Traversal for VoIP and SIP](https://oneuptime.com/blog/post/2026-03-20-nat-traversal-voip-sip/view)
- [How to Detect If You Are Behind Carrier-Grade NAT (CGNAT)](https://oneuptime.com/blog/post/2026-03-20-detect-cgnat/view)
