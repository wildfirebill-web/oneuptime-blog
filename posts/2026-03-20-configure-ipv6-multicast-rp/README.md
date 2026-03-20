# How to Configure IPv6 Multicast Rendezvous Points

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, PIM-SM, Rendezvous Point, Multicast, Network Routing

Description: A guide to configuring and managing IPv6 multicast Rendezvous Points (RP) for PIM-SM deployments, including static, BSR, and Anycast RP configurations.

## What Is a Rendezvous Point?

The Rendezvous Point (RP) is a router in a PIM-SM network where multicast sources register and receivers initially join. All multicast traffic flows through the RP until routers can switch to a direct source tree. For IPv6 multicast, the RP must have a global IPv6 address.

## Static RP Configuration

Static RP is the simplest approach - all routers are manually configured with the RP address:

```bash
# On all PIM routers in the domain (FRR vtysh)

vtysh
configure terminal

# Define the RP address for all multicast groups (or specific ranges)
ipv6 pim rp 2001:db8::rp ff3e::/32

# Or configure RP for all multicast groups
ipv6 pim rp 2001:db8::rp

# On the RP router itself (same command, plus interface config)
interface lo
 ipv6 address 2001:db8::rp/128
 ipv6 pim

end
write memory
```

## BSR (Bootstrap Router) for Dynamic RP

BSR (Bootstrap Router) protocol automatically distributes RP information to all PIM routers, eliminating the need for static configuration:

```bash
# On the BSR candidate router
vtysh
configure terminal

# Configure this router as a BSR candidate
ipv6 pim bsr candidate-bsr 2001:db8::bsr priority 100

# Configure RP candidates (routers that can serve as RP)
ipv6 pim bsr candidate-rp 2001:db8::rp group-list ff3e::/32 priority 10

# On all other routers: BSR is auto-discovered, but can specify
ipv6 pim bsr candidate-bsr 2001:db8::bsr2 priority 50  # lower priority = backup BSR

end
write memory
```

## Embedded RP (RFC 3956)

IPv6 has a unique feature called Embedded RP, where the RP address is embedded in the multicast group address itself. This eliminates the need for BSR or static RP configuration.

An embedded RP group address looks like:
```text
ff7<scope>:0<RP prefix length><RP prefix len>:<RP address low 32 bits>:<group ID>
```

Example:
```text
RP address: 2001:db8:1::1
Embedded RP group: ff7e:0240:2001:db8:1::1:beef
```

```bash
# FRR: enable embedded RP support
vtysh
configure terminal

# Enable embedded RP
ipv6 pim rp embedded

end
write memory

# Verify embedded RP is working
vtysh -c "show ipv6 pim rp-info"
```

## Anycast RP

For redundancy, multiple routers can share the same RP address (Anycast RP). When one RP fails, routing automatically shifts to the nearest surviving RP:

```bash
# Configure the same RP address on multiple routers
# RP Router 1
interface lo
 ipv6 address 2001:db8::rp/128  # shared anycast address

# RP Router 2 (different physical router, same anycast address)
interface lo
 ipv6 address 2001:db8::rp/128  # same anycast address

# Advertise the RP /128 via BGP or OSPF for reachability
# PIM will automatically use the nearest RP

# Configure MSDP (Multicast Source Discovery Protocol) between RP routers
# for source synchronization between Anycast RP instances
vtysh
configure terminal
router msdp
  peer 2001:db8::rp2 source 2001:db8::rp1
  peer 2001:db8::rp2 description "Anycast RP peer"
end
write memory
```

## Verifying RP Configuration

```bash
# Check RP information on all PIM routers
vtysh -c "show ipv6 pim rp-info"
# Expected:
# RP: 2001:db8::rp Group Prefix: ff3e::/32 Type: static

# Check BSR state
vtysh -c "show ipv6 pim bsr"

# Check which RP is being used for a specific group
vtysh -c "show ipv6 pim rp-info ff3e::stream"

# Verify source registration at the RP
vtysh -c "show ipv6 pim register-statistics"
```

## Testing RP with a Multicast Stream

```bash
# Source sends to a group in the RP's range
python3 -c "
import socket
s = socket.socket(socket.AF_INET6, socket.SOCK_DGRAM)
s.setsockopt(socket.IPPROTO_IPV6, socket.IPV6_MULTICAST_HOPS, 64)
for i in range(10):
    s.sendto(f'test {i}'.encode(), ('ff3e::db8:test', 5000))
    import time; time.sleep(1)
"

# Check that the RP sees the registration
vtysh -c "show ipv6 pim upstream"
# Look for (2001:db8::source, ff3e::db8:test) entry
```

## Summary

IPv6 multicast RPs can be configured statically (`ipv6 pim rp <addr>`), dynamically via BSR, or using IPv6's unique Embedded RP feature where the RP is encoded in the multicast address. For redundancy, use Anycast RP where multiple routers share the same RP address. Always verify RP configuration with `show ipv6 pim rp-info` and test with actual multicast sources and receivers.
