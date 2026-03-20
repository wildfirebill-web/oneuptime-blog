# How to Configure Dynamic NAT with an Address Pool

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Networking, NAT, IPv4, Cisco, Linux

Description: Learn how to configure dynamic NAT using a pool of public IP addresses to translate multiple private IP hosts.

## What Is Dynamic NAT?

Dynamic NAT translates private IP addresses to a pool of public IP addresses. Unlike static NAT (fixed 1:1), dynamic NAT assigns pool addresses on demand. When the pool is exhausted, new connections are dropped.

**Key difference from PAT**: Dynamic NAT is one-to-one (one private → one public at a time), but the mapping is not fixed. PAT maps many private IPs to one public IP using port numbers.

## Configuring Dynamic NAT on Cisco IOS

```cisco
! Step 1: Define the pool of public IPs
ip nat pool PUBLIC_POOL 203.0.113.10 203.0.113.20 netmask 255.255.255.0

! Step 2: Create ACL to match private IPs to translate
access-list 10 permit 192.168.1.0 0.0.0.255

! Step 3: Enable dynamic NAT
ip nat inside source list 10 pool PUBLIC_POOL

! Step 4: Mark interfaces
interface GigabitEthernet0/0
 ip nat inside

interface GigabitEthernet0/1
 ip nat outside
```

### Pool with Rotary Option

```cisco
! Round-robin assignment from the pool (for load balancing)
ip nat pool LB_POOL 203.0.113.10 203.0.113.20 prefix-length 24 type rotary
```

## Verifying Dynamic NAT on Cisco

```cisco
! View active NAT translations
show ip nat translations

! View statistics (hits, misses, expired)
show ip nat statistics

! Clear the translation table
clear ip nat translation *
```

Sample output:

```text
Pro Inside global      Inside local       Outside local      Outside global
tcp 203.0.113.10:1024  192.168.1.10:1024  8.8.8.8:80         8.8.8.8:80
tcp 203.0.113.11:1025  192.168.1.20:1025  1.1.1.1:443        1.1.1.1:443
```

## Configuring Dynamic NAT on Linux with iptables

Unlike Cisco's named pools, Linux NAT can use IP ranges:

```bash
# Translate 192.168.1.0/24 → pool of 203.0.113.10-203.0.113.20

iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o eth1 \
    -j SNAT --to-source 203.0.113.10-203.0.113.20
```

Linux will distribute source IPs across the pool.

## Dynamic NAT vs PAT Comparison

| Feature | Dynamic NAT | PAT (NAT Overload) |
|---------|-------------|-------------------|
| IP pool | Required (multiple IPs) | Single IP sufficient |
| Port translation | No | Yes |
| Concurrent connections | Limited by pool size | Very large (65535/IP) |
| Use case | Server farms | Home/office internet sharing |

## Key Takeaways

- Dynamic NAT assigns pool addresses on demand; no fixed mappings.
- The pool must have enough IPs to support concurrent connections.
- Exhausted pools silently drop new connections - use PAT for better scalability.
- On Linux, `--to-source IP1-IP2` specifies an IP range for dynamic SNAT.

**Related Reading:**

- [How to Configure Static NAT on a Router](https://oneuptime.com/blog/post/2026-03-20-configure-static-nat-router/view)
- [How to Configure PAT (Port Address Translation)](https://oneuptime.com/blog/post/2026-03-20-configure-pat-nat-overload/view)
- [How to Troubleshoot NAT Translation Issues](https://oneuptime.com/blog/post/2026-03-20-troubleshoot-nat-translation/view)
