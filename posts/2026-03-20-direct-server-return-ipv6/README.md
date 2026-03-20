# How to Implement Direct Server Return (DSR) with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, DSR, Direct Server Return, Load Balancing, LVS, Performance

Description: A guide to implementing Direct Server Return (DSR) load balancing with IPv6, where servers respond directly to clients bypassing the load balancer for outbound traffic.

Direct Server Return (DSR) is a load balancing mode where the load balancer only processes inbound traffic — servers respond directly to clients without the return traffic passing through the load balancer. This dramatically increases throughput since the load balancer only sees half the traffic. DSR with IPv6 requires careful NDP/ARP handling.

## DSR Architecture

```
                    ┌──────────────────────────────┐
                    │                              │
Client → Load Balancer → Server 1               Client
         (VIP: 2001:db8::vip)    ↗ (responds direct)
                     ↘
                       Server 2 → Client
                                  (responds direct)
```

The load balancer forwards packets to real servers with the destination VIP unchanged. Each server must have the VIP configured on loopback.

## Setup: Load Balancer (LVS/IPVS)

```bash
# On the load balancer: configure IPVS in DR mode
sudo ipvsadm -A -6 -t [2001:db8::vip]:80 -s rr

# Add real servers in DR (-g = gatewaying = DSR) mode
sudo ipvsadm -a -6 -t [2001:db8::vip]:80 -r [2001:db8::server1]:80 -g
sudo ipvsadm -a -6 -t [2001:db8::vip]:80 -r [2001:db8::server2]:80 -g

# Verify
sudo ipvsadm -L -n

# Enable IPv6 forwarding on load balancer
sudo sysctl -w net.ipv6.conf.all.forwarding=1
```

## Setup: Real Servers

Each real server must accept packets destined for the VIP:

```bash
# Add VIP to loopback interface on each server
sudo ip -6 addr add 2001:db8::vip/128 dev lo

# This is /128 (single address) — prevents NDP advertisements
# The server accepts packets to this address but doesn't respond to NDP for it

# Verify the address is on loopback
ip -6 addr show lo

# Test: server can receive traffic for VIP
# The packet arrives with dst=2001:db8::vip
# Server's loopback accepts it
# Server responds with src=2001:db8::server1 (or its own address)
# Client sees response from 2001:db8::server1, not the load balancer
```

## Suppress NDP for VIP on Real Servers

The critical DSR requirement: real servers must NOT respond to Neighbor Solicitation for the VIP (only the load balancer should respond):

```bash
# Method 1: Use /128 prefix on loopback (doesn't join solicited-node multicast)
sudo ip -6 addr add 2001:db8::vip/128 dev lo

# Method 2: Configure arp_ignore equivalent for IPv6
# Unfortunately Linux doesn't have ip6_ignore for NDP like it does for ARP
# The /128 on loopback approach is the standard solution

# Method 3: ip6tables to drop NS for VIP on non-loopback interfaces
sudo ip6tables -A INPUT -i eth0 \
  -p icmpv6 --icmpv6-type neighbor-solicitation \
  -d 2001:db8::vip \
  -j DROP
```

## DSR with HAProxy (IPv6)

HAProxy can also implement DSR-like behavior using transparent mode:

```
# /etc/haproxy/haproxy.cfg

global
    tune.bufsize 16384

frontend ipv6_frontend
    bind [2001:db8::vip]:80 v6only    # IPv6 only
    default_backend ipv6_servers

backend ipv6_servers
    balance roundrobin
    option tcp-check

    # source NAT disabled for DSR-like transparency
    server server1 [2001:db8::server1]:80 check
    server server2 [2001:db8::server2]:80 check
```

## Verifying DSR

```bash
# On load balancer: verify IPVS is forwarding
sudo ipvsadm -L -n --stats

# On client: send a request to VIP
curl -6 http://[2001:db8::vip]/

# On client: trace to see response comes from server, not LB
tcpdump -i eth0 -n 'host 2001:db8::vip or host 2001:db8::server1 or host 2001:db8::server2'

# DSR is working if:
# - Request: client→LB VIP
# - Response: server1→client (not LB→client)
```

## Performance Comparison

| Mode | LB handles | Throughput | Use case |
|---|---|---|---|
| NAT | Both directions | Limited by LB | Most deployments |
| DR (DSR) | Inbound only | Very high | High-bandwidth apps |
| TUN | Inbound only | High | Geographically distributed |

DSR with IPv6 is particularly valuable for high-bandwidth services like video streaming where response traffic is much larger than request traffic — the load balancer only sees the small inbound requests while servers send large responses directly to IPv6 clients.
