# How to Configure PAT (Port Address Translation) for IPv4 Address Conservation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: PAT, NAT, IPv4, iptables, nftables, Cisco IOS, Address Conservation, Masquerading

Description: Learn how to configure Port Address Translation (PAT/NAT overload) on Linux and Cisco IOS to allow multiple private IPv4 hosts to share a single public IP address.

---

PAT (also called NAT overload or IP masquerading) maps many private IPv4 addresses to a single public IP by using unique source port numbers. It is the most common form of NAT in home and enterprise networks.

## How PAT Works

```
Private (Inside)           | PAT Router           | Public (Outside)
192.168.1.10:45000  ──────►│ 203.0.113.1:10001  ──►│ Internet
192.168.1.20:45000  ──────►│ 203.0.113.1:10002  ──►│ Internet
192.168.1.30:45000  ──────►│ 203.0.113.1:10003  ──►│ Internet
```

## Linux: PAT with iptables (Masquerade)

```bash
# Enable IP forwarding
echo 1 > /proc/sys/net/ipv4/ip_forward
# Persist: add net.ipv4.ip_forward = 1 to /etc/sysctl.d/99-forward.conf

# PAT rule: masquerade all outbound traffic on the WAN interface (eth0)
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# Or use a specific public IP (SNAT) for predictable source address
iptables -t nat -A POSTROUTING -o eth0 -j SNAT --to-source 203.0.113.1

# Save rules
iptables-save > /etc/iptables/rules.v4
```

## Linux: PAT with nftables

```bash
# /etc/nftables.conf
table ip nat {
    chain postrouting {
        type nat hook postrouting priority srcnat;
        oifname "eth0" masquerade
    }
}

table ip filter {
    chain forward {
        type filter hook forward priority 0;
        ct state related,established accept
        iifname "eth1" oifname "eth0" accept
        drop
    }
}
```

## Cisco IOS: PAT (NAT Overload)

```
! Define inside and outside interfaces
interface GigabitEthernet0/0
  ip nat outside

interface GigabitEthernet0/1
  ip nat inside

! Create ACL matching private addresses to NAT
ip access-list standard PRIVATE-NETS
  permit 192.168.0.0 0.0.255.255
  permit 10.0.0.0 0.255.255.255
  permit 172.16.0.0 0.15.255.255

! Enable PAT (overload uses single public IP)
ip nat inside source list PRIVATE-NETS interface GigabitEthernet0/0 overload

! Verify
show ip nat translations
show ip nat statistics
```

## Verifying PAT on Linux

```bash
# Watch NAT table entries
conntrack -L -n     # requires conntrack-tools
# or
cat /proc/net/nf_conntrack | grep ESTABLISHED | head -20
```

## Key Takeaways

- PAT uses unique source port numbers to multiplex thousands of private connections through one public IP.
- On Linux, use `MASQUERADE` for dynamic IPs (like PPPoE/DHCP) or `SNAT` for fixed public IPs.
- Enable `net.ipv4.ip_forward = 1` in sysctl before configuring PAT — without it, forwarding is disabled.
- On Cisco IOS, the `overload` keyword converts NAT to PAT; without it, you need a 1:1 pool of public IPs.
