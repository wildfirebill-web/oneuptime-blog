# How to Configure LVS (Linux Virtual Server) for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: LVS, IPv6, Load Balancing, IPVS, Linux, High Performance

Description: A guide to configuring Linux Virtual Server (LVS) with IPVS for IPv6 load balancing, including NAT mode and DR mode configuration.

Linux Virtual Server (LVS) uses the kernel's IPVS (IP Virtual Server) framework to provide high-performance Layer 4 load balancing. IPVS supports IPv6 natively, enabling efficient IPv6 load balancing at kernel level without userspace overhead.

## Prerequisites

```bash
# Check if IPVS module is loaded

lsmod | grep ip_vs

# Load IPv6 IPVS modules
sudo modprobe ip_vs
sudo modprobe ip_vs_rr      # Round-robin scheduler
sudo modprobe ip_vs_wrr     # Weighted round-robin
sudo modprobe ip_vs_lc      # Least connections
sudo modprobe ip6table_filter

# Install ipvsadm
sudo apt-get install ipvsadm
```

## IPv6 LVS NAT Mode

In NAT mode, the load balancer translates the client destination IP to a real server IP:

```bash
# Enable IPv6 forwarding
sudo sysctl -w net.ipv6.conf.all.forwarding=1

# Add IPv6 virtual service (2001:db8::vip on port 80)
sudo ipvsadm -A -6 -t [2001:db8::vip]:80 -s rr

# Add IPv6 real servers
sudo ipvsadm -a -6 -t [2001:db8::vip]:80 -r [2001:db8::server1]:80 -m
sudo ipvsadm -a -6 -t [2001:db8::vip]:80 -r [2001:db8::server2]:80 -m

# -m = masquerading (NAT mode)
# -s rr = round-robin scheduler

# Verify configuration
sudo ipvsadm -L -n --stats
```

## IPv6 LVS Direct Routing (DR) Mode

In DR mode, the load balancer only handles inbound packets; responses go directly from servers to clients:

```bash
# On Load Balancer: set up virtual service in DR mode
sudo ipvsadm -A -6 -t [2001:db8::vip]:80 -s lc    # Least connections

sudo ipvsadm -a -6 -t [2001:db8::vip]:80 -r [2001:db8::server1]:80 -g
sudo ipvsadm -a -6 -t [2001:db8::vip]:80 -r [2001:db8::server2]:80 -g

# -g = gatewaying (DR mode)

# On each Real Server: add VIP to loopback (avoid ARP conflict)
sudo ip -6 addr add 2001:db8::vip/128 dev lo
# Suppress NDP announcements for this address
sudo ip -6 neigh add proxy 2001:db8::vip dev lo
```

## IPv6 LVS TUN Mode (IP Tunneling)

LVS Tunnel mode encapsulates packets in IPv6 tunnels:

```bash
# On Load Balancer
sudo ipvsadm -A -6 -t [2001:db8::vip]:80 -s wrr

sudo ipvsadm -a -6 -t [2001:db8::vip]:80 -r [2001:db8::server1]:80 -i -w 5
sudo ipvsadm -a -6 -t [2001:db8::vip]:80 -r [2001:db8::server2]:80 -i -w 3

# -i = ipip tunnel mode
# -w = weight
```

## UDP Load Balancing over IPv6

```bash
# UDP virtual service (e.g., DNS load balancing)
sudo ipvsadm -A -6 -u [2001:db8::dns-vip]:53 -s rr

sudo ipvsadm -a -6 -u [2001:db8::dns-vip]:53 -r [2001:db8::dns1]:53 -m
sudo ipvsadm -a -6 -u [2001:db8::dns-vip]:53 -r [2001:db8::dns2]:53 -m
```

## Persistence (Session Affinity)

```bash
# Enable connection persistence (sessions go to same server)
sudo ipvsadm -A -6 -t [2001:db8::vip]:443 -s rr -p 300

# -p 300 = 300 second persistence timeout
# All connections from same IPv6 client go to same server for 300 seconds
```

## Monitoring IPVS Statistics

```bash
# Show virtual services and real servers with statistics
sudo ipvsadm -L -n --stats

# Watch in real-time
watch -n 2 'sudo ipvsadm -L -n --stats'

# Show connection table
sudo ipvsadm -Lnc | grep "2001:"

# Check IPVS via /proc
cat /proc/net/ip_vs
cat /proc/net/ip6_vs_conn   # Active IPv6 connections
```

## Making Configuration Persistent

```bash
# Save IPVS rules
sudo ipvsadm --save > /etc/ipvsadm.rules

# Restore on boot (add to /etc/rc.local or systemd service)
sudo ipvsadm --restore < /etc/ipvsadm.rules

# Or use keepalived to manage LVS + health checking
sudo apt-get install keepalived
```

LVS/IPVS's kernel-level IPv6 load balancing provides the highest performance option for Layer 4 IPv6 traffic distribution, capable of millions of concurrent connections with minimal CPU overhead.
