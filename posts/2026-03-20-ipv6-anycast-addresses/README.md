# How to Understand IPv6 Anycast Addresses

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Networking, Anycast, Routing, High Availability

Description: Understand IPv6 anycast addresses, how they differ from unicast and multicast, and practical use cases including DNS and CDN deployments.

## Introduction

Anycast is one of three addressing modes in IPv6 (alongside unicast and multicast). An anycast address is assigned to multiple interfaces on different nodes; packets sent to an anycast address are delivered to the topologically nearest node that has that address. Unlike multicast, only one node receives the packet — the closest one according to the routing protocol.

## How Anycast Works

```mermaid
graph TD
    Client -->|sends to 2001:db8::1| Router
    Router -->|routing table| BestPath{Nearest node?}
    BestPath -->|Node A - 2 hops| ServerA[Server A<br/>2001:db8::1]
    BestPath -->|Node B - 5 hops| ServerB[Server B<br/>2001:db8::1]
    BestPath -->|Node C - 8 hops| ServerC[Server C<br/>2001:db8::1]
    style ServerA fill:#4CAF50,color:#fff
```

All servers advertise the same prefix. The routing protocol (BGP, OSPF) selects the best path, directing traffic to the nearest server automatically.

## Anycast vs Unicast vs Multicast

| Property | Unicast | Anycast | Multicast |
|---|---|---|---|
| Receivers | One specific node | One nearest node | All group members |
| Address assignment | One interface | Multiple interfaces | Group subscription |
| Use case | Point-to-point | Load balancing, HA | Group communication |
| IPv6 requirement | Yes | Yes | Yes |

## Subnet-Router Anycast Address

IPv6 defines a mandatory anycast address for every subnet — the **Subnet-Router Anycast Address**. It is the lowest address in the subnet (all interface ID bits set to zero):

```
Subnet prefix: 2001:db8:1:2::/64
Subnet-Router anycast: 2001:db8:1:2::
```

Routers must support this address so that mobile nodes can send packets to any router on the subnet without knowing the specific router address.

## Configuring Anycast on Linux

To configure an anycast address on Linux, use the `anycast` flag:

```bash
# Add an anycast address to an interface
# The 'anycast' keyword tells the kernel this is an anycast address
sudo ip -6 addr add 2001:db8::1/128 anycast dev eth0

# Verify the anycast address
ip -6 addr show dev eth0

# Check routing table entries
ip -6 route show

# Remove an anycast address
sudo ip -6 addr del 2001:db8::1/128 anycast dev eth0
```

## BGP Anycast for DNS

The most common real-world use of anycast is DNS. All major DNS providers (Cloudflare's 2606:4700:4700::1111, Google's 2001:4860:4860::8888) use BGP anycast:

```bash
# Cloudflare's IPv6 DNS anycast addresses
# 2606:4700:4700::1111 and 2606:4700:4700::1001
# These are announced from dozens of PoPs worldwide

# Check which PoP you're hitting via traceroute
traceroute6 2606:4700:4700::1111

# Measure latency to anycast DNS
ping6 -c 5 2606:4700:4700::1111
```

## Practical Anycast Use Cases

1. **DNS resolution** — Multiple DNS servers share one address; clients always hit the nearest one
2. **CDN edge nodes** — Content served from the nearest geographic location
3. **NTP services** — pool.ntp.org uses anycast for time synchronization
4. **DDoS mitigation** — Distribute attack traffic across many nodes

## Anycast with OSPFv3 (Internal Anycast)

For internal anycast deployments using OSPFv3:

```
# On each anycast server, add the address as a loopback:
# /etc/network/interfaces (Debian/Ubuntu)
auto lo:anycast
iface lo:anycast inet6 static
    address 2001:db8:service::1
    netmask 128

# Then redistribute into OSPFv3 so all routers learn the route
# Each server's nearest router will advertise the /128
```

## Limitations and Considerations

- **TCP sessions**: Anycast works well for stateless protocols (UDP/DNS). For TCP, session persistence can be broken if routing changes mid-session.
- **No source address selection**: Hosts cannot distinguish anycast addresses from regular unicast; they send normally.
- **Routing convergence**: If an anycast node fails, routing convergence time determines how quickly traffic shifts.

## Conclusion

IPv6 anycast is a powerful mechanism for building resilient, geographically distributed services. It is most effective for short-lived, stateless interactions like DNS queries. Understanding anycast helps network engineers design high-availability services that automatically route clients to the nearest healthy node without client-side configuration.
