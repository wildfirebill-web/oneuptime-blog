# How to Set Up NAT Masquerading on a Linux Gateway

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Networking, NAT, Linux, Gateway, IPv4

Description: Learn how to configure a Linux machine as a NAT gateway using MASQUERADE so all LAN clients can share a single internet connection.

## What Is NAT Masquerading?

MASQUERADE is a special form of SNAT that automatically uses the current IP address of the outgoing interface as the source IP. It is ideal for gateways with dynamic (DHCP-assigned) public IPs.

## Gateway Setup

```text
[LAN Clients]          [Linux Gateway]          [Internet]
192.168.1.0/24   eth0(192.168.1.1)  eth1(DHCP)  8.8.8.8
```

## Step 1: Assign Addresses and Interfaces

```bash
# Confirm interface assignments

ip addr show eth0  # LAN interface: 192.168.1.1/24
ip addr show eth1  # WAN interface: DHCP address
```

## Step 2: Enable IP Forwarding

```bash
# Temporary
echo 1 > /proc/sys/net/ipv4/ip_forward

# Permanent
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
sysctl -p
```

## Step 3: Configure MASQUERADE

```bash
# NAT all LAN traffic going out eth1
iptables -t nat -A POSTROUTING -o eth1 -j MASQUERADE

# Allow LAN to internet
iptables -A FORWARD -i eth0 -o eth1 -j ACCEPT

# Allow established/related return traffic
iptables -A FORWARD -i eth1 -o eth0 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

## Step 4: Configure LAN Clients

On each client, set:
- IP: 192.168.1.x/24
- Gateway: 192.168.1.1 (the Linux gateway)
- DNS: 8.8.8.8 or your local resolver

```bash
# On a client (Linux):
ip addr add 192.168.1.10/24 dev eth0
ip route add default via 192.168.1.1
echo "nameserver 8.8.8.8" > /etc/resolv.conf
```

## Step 5: Add DNS Service (Optional)

Run dnsmasq on the gateway for DHCP and DNS:

```bash
apt install dnsmasq

cat > /etc/dnsmasq.conf << 'CONF'
interface=eth0
dhcp-range=192.168.1.100,192.168.1.200,24h
dhcp-option=3,192.168.1.1
dhcp-option=6,8.8.8.8
CONF

systemctl restart dnsmasq
```

## Complete Startup Script

```bash
#!/bin/bash
# /etc/network/nat-gateway.sh

# Flush existing rules
iptables -F
iptables -t nat -F

# Enable forwarding
sysctl -w net.ipv4.ip_forward=1

# NAT masquerade
iptables -t nat -A POSTROUTING -o eth1 -j MASQUERADE

# Forwarding rules
iptables -A FORWARD -i eth0 -o eth1 -j ACCEPT
iptables -A FORWARD -i eth1 -o eth0 -m state --state RELATED,ESTABLISHED -j ACCEPT

# Allow loopback
iptables -A INPUT -i lo -j ACCEPT

echo "NAT gateway configured"
```

Make it run at boot:

```bash
# Ubuntu: add to /etc/rc.local or use systemd
chmod +x /etc/network/nat-gateway.sh
echo "@reboot root /etc/network/nat-gateway.sh" >> /etc/crontab
```

## Verifying the Gateway

```bash
# From a LAN client, test internet access
ping 8.8.8.8
curl https://ifconfig.me   # Should show gateway's public IP

# On the gateway, confirm NAT is working
conntrack -L | head -10
iptables -t nat -L -n -v
```

## Using nftables Instead

```bash
#!/usr/sbin/nft -f
flush ruleset

table inet filter {
    chain forward {
        type filter hook forward priority 0; policy drop;
        ct state related,established accept
        iifname "eth0" oifname "eth1" accept
    }
}

table inet nat {
    chain postrouting {
        type nat hook postrouting priority 100;
        oifname "eth1" masquerade
    }
}
```

## Key Takeaways

- MASQUERADE automatically uses the outgoing interface's current IP as source.
- IP forwarding (`net.ipv4.ip_forward=1`) is required for packet forwarding.
- Always add FORWARD rules to allow LAN→WAN and established WAN→LAN traffic.
- Add dnsmasq for a complete DHCP+DNS+NAT gateway solution.

**Related Reading:**

- [How to Configure NAT on Linux Using iptables](https://oneuptime.com/blog/post/2026-03-20-nat-linux-iptables/view)
- [How to Configure Source NAT (SNAT) on Linux](https://oneuptime.com/blog/post/2026-03-20-configure-snat-linux/view)
- [How to Set Up IP Forwarding on Linux](https://oneuptime.com/blog/post/2026-03-20-ip-forwarding-linux/view)
