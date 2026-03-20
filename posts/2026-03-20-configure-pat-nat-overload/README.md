# How to Configure PAT (Port Address Translation) / NAT Overload

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Networking, NAT, PAT, IPv4, Linux, Cisco

Description: Learn how to configure PAT (also called NAT Overload) to allow many private hosts to share a single public IP address using port translation.

## What Is PAT?

PAT (Port Address Translation), also called NAT Overload, maps multiple private IP addresses to a single public IP by using different port numbers. This is the most common form of NAT used in homes and offices.

```
192.168.1.10:54321 → 203.0.113.1:1024  (toward 8.8.8.8:80)
192.168.1.20:54322 → 203.0.113.1:1025  (toward 8.8.8.8:80)
192.168.1.30:54323 → 203.0.113.1:1026  (toward 1.1.1.1:443)
```

## Configuring PAT on Cisco IOS

### Using the Outside Interface IP

```cisco
! ACL to match inside hosts
access-list 10 permit 192.168.1.0 0.0.0.255

! PAT using the outside interface IP (overload keyword)
ip nat inside source list 10 interface GigabitEthernet0/1 overload

! Mark interfaces
interface GigabitEthernet0/0
 ip nat inside

interface GigabitEthernet0/1
 ip nat outside
```

### Using a Static Pool with Overload

```cisco
ip nat pool PAT_POOL 203.0.113.1 203.0.113.1 netmask 255.255.255.0
ip nat inside source list 10 pool PAT_POOL overload
```

## Configuring PAT on Linux with iptables MASQUERADE

For dynamic external IPs (DHCP):

```bash
# Enable forwarding
echo 1 > /proc/sys/net/ipv4/ip_forward

# MASQUERADE: automatically uses current IP of eth1
iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o eth1 -j MASQUERADE

# Allow forwarded traffic
iptables -A FORWARD -i eth0 -o eth1 -j ACCEPT
iptables -A FORWARD -i eth1 -o eth0 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

## Configuring PAT on Linux with iptables SNAT

For static external IPs (preferred for servers):

```bash
# SNAT with fixed public IP
iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o eth1 \
    -j SNAT --to-source 203.0.113.1
```

## Configuring PAT with nftables

```bash
table inet nat {
    chain postrouting {
        type nat hook postrouting priority 100;
        ip saddr 192.168.1.0/24 oifname "eth1" masquerade
    }
}

# Apply
nft -f /etc/nftables.conf
```

## Making iptables Rules Persistent

```bash
# Ubuntu/Debian
sudo apt install iptables-persistent
sudo netfilter-persistent save

# RHEL/CentOS
sudo service iptables save
```

## Verifying PAT

```bash
# On Linux: show current NAT translations
sudo conntrack -L | head -20

# Count active connections
sudo conntrack -L | wc -l

# On Cisco: show translations
show ip nat translations

# Test from an inside host
curl -v http://ifconfig.me
# Should return the public IP (203.0.113.1)
```

## PAT Port Exhaustion

PAT supports up to ~65,535 connections per public IP. For very high-traffic environments:

```bash
# Use multiple public IPs with overload
iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o eth1 \
    -j SNAT --to-source 203.0.113.1-203.0.113.10
```

## Key Takeaways

- PAT maps many private IPs to one public IP using source port translation.
- On Cisco, add the `overload` keyword to `ip nat inside source` to enable PAT.
- On Linux, use MASQUERADE for dynamic IPs or SNAT for static public IPs.
- PAT supports ~65K concurrent sessions per public IP address.

**Related Reading:**

- [How to Configure Static NAT on a Router](https://oneuptime.com/blog/post/2026-03-20-configure-static-nat-router/view)
- [How to Set Up Port Forwarding with NAT](https://oneuptime.com/blog/post/2026-03-20-port-forwarding-nat/view)
- [How to Configure NAT on Linux Using iptables](https://oneuptime.com/blog/post/2026-03-20-nat-linux-iptables/view)
