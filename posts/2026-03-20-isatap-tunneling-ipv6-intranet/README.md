# How to Configure ISATAP Tunneling for IPv6 on IPv4 Intranets

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ISATAP, IPv6, Tunneling, IPv4, Intranet, Transition

Description: Configure ISATAP (Intra-Site Automatic Tunnel Addressing Protocol) to provide IPv6 connectivity to IPv4-only intranet hosts without upgrading the network infrastructure.

## Introduction

ISATAP (RFC 5214) allows IPv6 hosts to communicate over IPv4 intranets by embedding IPv4 addresses in the lower 32 bits of IPv6 addresses (prefix::0:5efe:a.b.c.d format). It requires an ISATAP router that has both IPv4 and IPv6 connectivity.

## ISATAP Address Format

```
ISATAP address: <64-bit prefix>::0:5efe:a.b.c.d
                <64-bit prefix>::200:5efe:a.b.c.d  (globally unique)
                <64-bit prefix>::0:5efe:10.0.0.5

For host 10.0.0.5 with prefix 2001:db8::/64:
IPv6 address: 2001:db8::0:5efe:a00:5
(10.0.0.5 in hex = 0a.00.00.05 = a00:005 — wait: 10=0a, 0=00, 0=00, 5=05)
Full: 2001:db8::5efe:a00:5
```

## Linux ISATAP Host Configuration

```bash
# Create ISATAP tunnel interface
sudo ip tunnel add isatap0 mode isatap local 10.0.0.5 ttl 64

# Bring interface up
sudo ip link set isatap0 up

# Assign IPv6 address using ISATAP format
# Using prefix 2001:db8::/64 and IPv4 10.0.0.5
sudo ip address add 2001:db8::5efe:a00:5/64 dev isatap0

# Add route to IPv6 network via ISATAP router
# ISATAP router's IPv4 address: 10.0.0.1
sudo ip -6 route add ::/0 via ::5efe:a00:1 dev isatap0
```

## ISATAP Router Configuration

```bash
# The ISATAP router has both IPv4 and native IPv6

# Create ISATAP tunnel (router-side)
sudo ip tunnel add isatap-router mode isatap local 10.0.0.1 ttl 64
sudo ip link set isatap-router up
sudo ip address add 2001:db8::5efe:a00:1/64 dev isatap-router

# Enable forwarding
sudo sysctl -w net.ipv6.conf.all.forwarding=1

# Route ISATAP subnet to ISATAP interface
sudo ip -6 route add 2001:db8::/64 dev isatap-router

# The router must respond to Potential Router Solicitations
# Configure radvd to advertise on the ISATAP interface:
```

```bash
# /etc/radvd.conf (on ISATAP router)
interface isatap-router {
    AdvSendAdvert on;
    AdvDefaultLifetime 1800;
    prefix 2001:db8::/64 {
        AdvOnLink off;          # Not on-link (ISATAP is host-to-router)
        AdvAutonomous on;
    };
};
```

## DNS Configuration for ISATAP

```bash
# Add a DNS A record for "isatap" pointing to the ISATAP router
# isatap.example.com.  IN A  10.0.0.1

# Windows ISATAP hosts automatically query DNS for "isatap.domain"
# to find the ISATAP router

# Linux hosts need manual router configuration (above)
```

## Persistent Configuration

```bash
# /etc/network/interfaces (Debian)
auto isatap0
iface isatap0 inet6 tunnel
    address 2001:db8::5efe:a00:5
    netmask 64
    endpoint any
    local 10.0.0.5
    mode isatap
    gateway ::5efe:a00:1
```

## Testing ISATAP

```bash
# Verify ISATAP interface
ip address show isatap0

# Test connectivity to ISATAP router
ping -6 2001:db8::5efe:a00:1

# Test connectivity to another ISATAP host (10.0.0.6)
ping -6 2001:db8::5efe:a00:6

# Trace the path
traceroute6 2001:db8::5efe:a00:1
```

## Conclusion

ISATAP embeds IPv4 addresses into IPv6 addresses using the `::5efe:` marker, allowing IPv6 communication over IPv4 intranets. Configure an ISATAP router (a dual-stack host) that advertises the IPv6 prefix, and add ISATAP tunnel interfaces on each host. ISATAP is primarily used in enterprise intranet scenarios for gradual IPv6 deployment without replacing IPv4 infrastructure. For new deployments, native dual-stack is preferred.
