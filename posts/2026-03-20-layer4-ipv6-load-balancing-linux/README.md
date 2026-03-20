# How to Configure Layer 4 IPv6 Load Balancing on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Layer 4, Load Balancing, Linux, IPVS, ip6tables, Networking

Description: A guide to configuring Layer 4 IPv6 load balancing on Linux using IPVS, nftables, and ip6tables for TCP and UDP traffic distribution.

Layer 4 load balancing distributes TCP and UDP traffic based on source/destination IP and port without inspecting application-layer content. Linux provides multiple approaches for IPv6 Layer 4 load balancing: IPVS (highest performance), nftables (modern and flexible), and HAProxy in TCP mode.

## Method 1: IPVS (Best Performance)

IPVS operates in the kernel and provides the highest performance:

```bash
# Load required modules
sudo modprobe ip_vs ip_vs_rr ip_vs_wrr ip_vs_lc

# TCP load balancing (round-robin)
sudo ipvsadm -A -6 -t [2001:db8::vip]:443 -s rr

sudo ipvsadm -a -6 -t [2001:db8::vip]:443 -r [2001:db8::server1]:443 -m
sudo ipvsadm -a -6 -t [2001:db8::vip]:443 -r [2001:db8::server2]:443 -m

# UDP load balancing (DNS)
sudo ipvsadm -A -6 -u [2001:db8::dns-vip]:53 -s wrr

sudo ipvsadm -a -6 -u [2001:db8::dns-vip]:53 -r [2001:db8::ns1]:53 -m -w 2
sudo ipvsadm -a -6 -u [2001:db8::dns-vip]:53 -r [2001:db8::ns2]:53 -m -w 1

# View statistics
sudo ipvsadm -L -n --stats
```

## Method 2: nftables with Load Balancing

nftables supports `numgen` for round-robin distribution and `jhash` for consistent hashing:

```bash
# Create nftables load balancing rules
sudo nft -f - << 'EOF'
table ip6 lb {
    chain prerouting {
        type nat hook prerouting priority dstnat;

        # Round-robin across 3 servers
        ip6 daddr 2001:db8::vip tcp dport 80 \
            dnat to numgen random mod 3 map {
                0 : [2001:db8::server1]:80,
                1 : [2001:db8::server2]:80,
                2 : [2001:db8::server3]:80
            }
    }

    chain postrouting {
        type nat hook postrouting priority srcnat;
        # Masquerade for NAT mode
        ip6 saddr 2001:db8::/64 oifname "eth0" masquerade
    }
}
EOF
```

## Method 3: Consistent Hash Load Balancing (nftables)

Consistent hashing ensures the same client always goes to the same server:

```bash
sudo nft -f - << 'EOF'
table ip6 lb_hash {
    chain prerouting {
        type nat hook prerouting priority dstnat;

        # Hash based on source IP (same client → same server)
        ip6 daddr 2001:db8::vip tcp dport 80 \
            dnat to jhash ip6 saddr mod 2 map {
                0 : [2001:db8::server1]:80,
                1 : [2001:db8::server2]:80
            }
    }
}
EOF
```

## Method 4: ip6tables DNAT

```bash
# Enable IPv6 forwarding
sudo sysctl -w net.ipv6.conf.all.forwarding=1

# DNAT to alternate between two servers using statistic module
sudo ip6tables -t nat -A PREROUTING \
  -d 2001:db8::vip -p tcp --dport 80 \
  -m statistic --mode nth --every 2 --packet 0 \
  -j DNAT --to-destination [2001:db8::server1]:80

sudo ip6tables -t nat -A PREROUTING \
  -d 2001:db8::vip -p tcp --dport 80 \
  -j DNAT --to-destination [2001:db8::server2]:80

# Masquerade for return traffic
sudo ip6tables -t nat -A POSTROUTING \
  -s 2001:db8::/64 -o eth0 \
  -j MASQUERADE
```

## Method 5: HAProxy (TCP Mode)

```
# /etc/haproxy/haproxy.cfg

global
    daemon
    maxconn 50000

defaults
    mode tcp
    timeout connect 5s
    timeout client 30s
    timeout server 30s

frontend ipv6_frontend
    bind [2001:db8::vip]:443 v6only
    default_backend ipv6_backend

backend ipv6_backend
    balance roundrobin
    option tcp-check
    server s1 [2001:db8::server1]:443 check
    server s2 [2001:db8::server2]:443 check
    server s3 [2001:db8::server3]:443 check
```

## Choosing the Right Method

| Method | Throughput | Flexibility | Health Check | Complexity |
|---|---|---|---|---|
| IPVS | Highest (kernel) | Medium | External needed | Low |
| nftables | High (kernel) | High | External needed | Medium |
| ip6tables | High (kernel) | Medium | External needed | Low |
| HAProxy TCP | Medium | High | Built-in | Low |

For maximum throughput: IPVS + keepalived (for VIP failover + health checking)
For flexibility: nftables
For simplicity: HAProxy

Layer 4 IPv6 load balancing on Linux is well-supported across all methods, with IPVS providing the best performance for high-traffic scenarios.
