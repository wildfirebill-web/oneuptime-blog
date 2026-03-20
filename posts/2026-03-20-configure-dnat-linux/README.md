# How to Configure Destination NAT (DNAT) on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Networking, NAT, DNAT, Linux, iptables

Description: Learn how to configure Destination NAT (DNAT) on Linux using iptables and nftables to redirect incoming traffic to different hosts or ports.

## What Is DNAT?

Destination NAT (DNAT) modifies the **destination IP address** (and optionally port) of incoming packets. It is applied in the PREROUTING chain, before routing decisions are made.

**Use cases:**
- Port forwarding to internal servers
- Load balancing across backend servers
- Transparent proxying
- Redirecting traffic to a local process

## Basic DNAT with iptables

```bash
# Enable IP forwarding

echo 1 > /proc/sys/net/ipv4/ip_forward

# Forward incoming port 80 to 192.168.1.10:80
iptables -t nat -A PREROUTING -i eth1 -p tcp --dport 80 \
    -j DNAT --to-destination 192.168.1.10:80

# Allow the forwarded traffic in the FORWARD chain
iptables -A FORWARD -i eth1 -p tcp -d 192.168.1.10 --dport 80 -j ACCEPT
```

## DNAT to a Different Port

```bash
# External 2222 → Internal 192.168.1.20:22 (SSH)
iptables -t nat -A PREROUTING -i eth1 -p tcp --dport 2222 \
    -j DNAT --to-destination 192.168.1.20:22

iptables -A FORWARD -i eth1 -p tcp -d 192.168.1.20 --dport 22 -j ACCEPT
```

## DNAT Based on Destination IP

```bash
# Only DNAT traffic arriving for specific public IP
iptables -t nat -A PREROUTING -i eth1 \
    -d 203.0.113.10 -p tcp --dport 80 \
    -j DNAT --to-destination 192.168.1.10:80
```

## DNAT with Port Range

```bash
# Forward ports 8000-8010 to backend
iptables -t nat -A PREROUTING -i eth1 -p tcp --dport 8000:8010 \
    -j DNAT --to-destination 192.168.1.10
```

## DNAT for Local Redirect (OUTPUT Chain)

For locally-generated traffic (not arriving on an external interface):

```bash
# Redirect outgoing traffic to port 80 to a local transparent proxy (port 3128)
iptables -t nat -A OUTPUT -p tcp --dport 80 \
    -j DNAT --to-destination 127.0.0.1:3128
```

## DNAT with nftables

```bash
table inet nat {
    chain prerouting {
        type nat hook prerouting priority -100;
        
        # Basic port forward
        iifname "eth1" tcp dport 80 dnat to 192.168.1.10:80
        
        # External port 2222 → internal SSH
        iifname "eth1" tcp dport 2222 dnat to 192.168.1.20:22
        
        # Multiple ports
        iifname "eth1" tcp dport { 80, 443 } dnat to 192.168.1.10
    }
}
```

## Transparent Proxy with DNAT

```bash
# Redirect all HTTP traffic from LAN to local Squid proxy
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 80 \
    -j DNAT --to-destination 192.168.1.1:3128

# Exclude local traffic
iptables -t nat -A PREROUTING -i eth0 -s 192.168.1.1 -p tcp --dport 80 -j RETURN
```

## Verifying DNAT Rules

```bash
# List PREROUTING chain
iptables -t nat -L PREROUTING -n -v

# Test port forward from external
nc -zv 203.0.113.1 80
curl http://203.0.113.1

# View active DNAT connections
conntrack -L | grep DNAT
```

## Key Takeaways

- DNAT modifies destination IP/port in PREROUTING before routing.
- Always add a FORWARD rule to allow the DNAT'd traffic to pass.
- nftables DNAT syntax: `dnat to IP:port` or `dnat to IP`.
- DNAT in OUTPUT chain redirects locally generated traffic.

**Related Reading:**

- [How to Configure Source NAT (SNAT) on Linux](https://oneuptime.com/blog/post/2026-03-20-configure-snat-linux/view)
- [How to Set Up Port Forwarding with NAT](https://oneuptime.com/blog/post/2026-03-20-port-forwarding-nat/view)
- [How to Configure NAT on Linux Using iptables](https://oneuptime.com/blog/post/2026-03-20-nat-linux-iptables/view)
