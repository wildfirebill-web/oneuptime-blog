# How to Configure Static NAT on a Router

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Networking, NAT, IPv4, Cisco, Linux

Description: Learn how to configure static NAT to create a one-to-one mapping between a private IP address and a public IP address on Cisco and Linux routers.

## What Is Static NAT?

Static NAT creates a permanent one-to-one mapping between a private (inside local) IP address and a public (inside global) IP address. Unlike dynamic NAT, the mapping does not change.

**Use case**: Hosting a public-facing server (web, mail, FTP) that always needs the same public IP.

## Static NAT Terminology

| Term | Description | Example |
|------|-------------|---------|
| Inside Local | Private IP of internal host | 192.168.1.10 |
| Inside Global | Public IP visible to outside world | 203.0.113.10 |
| Outside Local | IP used to reach external host from inside | 8.8.8.8 |
| Outside Global | Actual external host IP | 8.8.8.8 |

## Configuring Static NAT on Cisco IOS

```cisco
! Define inside and outside interfaces
interface GigabitEthernet0/0
 ip address 192.168.1.1 255.255.255.0
 ip nat inside

interface GigabitEthernet0/1
 ip address 203.0.113.1 255.255.255.0
 ip nat outside

! Create static NAT mapping
ip nat inside source static 192.168.1.10 203.0.113.10

! Verify
show ip nat translations
show ip nat statistics
```

### Multiple Static NAT Entries

```cisco
ip nat inside source static 192.168.1.10 203.0.113.10
ip nat inside source static 192.168.1.20 203.0.113.20
ip nat inside source static 192.168.1.30 203.0.113.30
```

### Bidirectional Traffic

Static NAT works in both directions:
- Inside → Outside: 192.168.1.10 appears as 203.0.113.10
- Outside → Inside: Traffic to 203.0.113.10 is forwarded to 192.168.1.10

## Configuring Static NAT on Linux with iptables

```bash
# Enable IP forwarding

echo 1 > /proc/sys/net/ipv4/ip_forward

# eth0 = inside (private), eth1 = outside (public)

# DNAT: Incoming traffic to public IP → forward to private IP
iptables -t nat -A PREROUTING -i eth1 -d 203.0.113.10 -j DNAT --to-destination 192.168.1.10

# SNAT: Outgoing traffic from private IP → appear as public IP
iptables -t nat -A POSTROUTING -o eth1 -s 192.168.1.10 -j SNAT --to-source 203.0.113.10

# Allow forwarded traffic
iptables -A FORWARD -i eth1 -o eth0 -d 192.168.1.10 -j ACCEPT
iptables -A FORWARD -i eth0 -o eth1 -s 192.168.1.10 -j ACCEPT
```

## Configuring Static NAT on Linux with nftables

```bash
table inet nat {
    chain prerouting {
        type nat hook prerouting priority -100;
        ip daddr 203.0.113.10 dnat to 192.168.1.10
    }
    chain postrouting {
        type nat hook postrouting priority 100;
        ip saddr 192.168.1.10 oifname "eth1" snat to 203.0.113.10
    }
}
```

## Verifying Static NAT

```bash
# Linux: Check NAT rules
iptables -t nat -L -n -v

# Check active connections
conntrack -L | grep 203.0.113.10

# From external host, test connectivity to public IP
curl http://203.0.113.10
```

## Key Takeaways

- Static NAT creates a permanent one-to-one IP mapping.
- Traffic flows in both directions - inbound and outbound.
- On Cisco: `ip nat inside source static private_ip public_ip`
- On Linux: Combine DNAT (PREROUTING) and SNAT (POSTROUTING) for bidirectional static NAT.

**Related Reading:**

- [How to Configure Dynamic NAT with an Address Pool](https://oneuptime.com/blog/post/2026-03-20-configure-dynamic-nat-address-pool/view)
- [How to Configure PAT (Port Address Translation)](https://oneuptime.com/blog/post/2026-03-20-configure-pat-nat-overload/view)
- [How to Understand Inside Local, Inside Global, Outside Local, Outside Global](https://oneuptime.com/blog/post/2026-03-20-nat-inside-outside-local-global/view)
