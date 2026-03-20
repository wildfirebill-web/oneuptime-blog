# How to Configure DS-Lite with AFTR on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, DS-Lite, AFTR, Linux, ISP Configuration

Description: Step-by-step instructions for configuring an AFTR (Address Family Transition Router) for DS-Lite on Linux to terminate IPv4-in-IPv6 Softwire tunnels from B4 CPE devices.

## Prerequisites

- Linux server with an IPv6 address reachable by B4 devices
- Public IPv4 address pool for CGN NAT44
- Kernel support for IPv6 tunneling (`ip6_tunnel` module)

## Understanding AFTR's Role

The AFTR performs two functions:
1. **Softwire tunnel termination**: Decapsulates IPv4-in-IPv6 packets from B4 devices
2. **CGN NAT44**: Translates subscriber private IPv4 addresses to shared public addresses

## Method 1: Using lwAFTR (Snabb-based AFTR)

For production deployments, Snabb's lwAFTR (Lightweight 4over6 AFTR) offers high-performance packet processing. For lab/testing, use Linux's built-in `ip6tnl` tunnels with iptables NAT.

## Method 2: Linux ip6tnl + NAT44 (Lab/Small Scale)

### Step 1: Load Required Kernel Modules

```bash
# Load IPv6 tunnel module
modprobe ip6_tunnel
modprobe ip6table_mangle

# Verify modules are loaded
lsmod | grep ip6_tunnel
```

### Step 2: Create the Softwire Tunnel Interface

For DS-Lite, each B4 has a dedicated tunnel. For simplicity, we configure a generic `any-to-any` tunnel:

```bash
# Create an ip6tnl tunnel that accepts from any B4 IPv6 address
# Mode ip4ip6 = encapsulated IPv4 over IPv6
ip tunnel add aftr0 mode ip4ip6 \
    local 2001:db8::aftr \
    remote any \
    encaplimit none \
    dev eth0

# Bring the tunnel interface up
ip link set aftr0 up

# Assign an internal IPv4 address to the tunnel interface
ip addr add 192.0.0.1/24 dev aftr0
```

### Step 3: Configure CGN NAT44

All IPv4 traffic arriving from B4 devices (after decapsulation) must be NATted to public addresses:

```bash
# NAT all decapsulated IPv4 traffic to the public IPv4 address pool
# Replace 203.0.113.0/28 with your actual public IPv4 addresses
iptables -t nat -A POSTROUTING -o eth1 -s 10.0.0.0/8 -j SNAT \
    --to-source 203.0.113.1-203.0.113.14

# Alternatively, use MASQUERADE if the public IP is dynamic
iptables -t nat -A POSTROUTING -o eth1 -s 10.0.0.0/8 -j MASQUERADE

# Enable IPv4 forwarding
sysctl -w net.ipv4.ip_forward=1

# Enable IPv6 forwarding (needed for tunnel traffic)
sysctl -w net.ipv6.conf.all.forwarding=1
```

### Step 4: Add Routes for Subscriber Networks

```bash
# Route traffic from each B4's IPv4 space through the tunnel
# If all B4s use RFC 1918 space, a single route covers them
ip route add 10.0.0.0/8 dev aftr0
ip route add 172.16.0.0/12 dev aftr0
ip route add 192.168.0.0/16 dev aftr0
```

## Configuring the B4 Side (CPE/Router)

On the home router or B4 device:

```bash
# Create an IPv4-in-IPv6 tunnel from B4 to AFTR
ip tunnel add b4tun0 mode ip4ip6 \
    local 2001:db8:subscriber::1 \
    remote 2001:db8::aftr \
    dev eth0

ip link set b4tun0 up

# Assign a private IPv4 address to the tunnel
ip addr add 192.168.100.1/24 dev b4tun0

# Route all IPv4 traffic through the B4 tunnel to AFTR
ip route add default dev b4tun0 mtu 1460

# Set MTU to account for IPv6 tunnel overhead (40 bytes)
ip link set b4tun0 mtu 1460
```

## Configuring AFTR Discovery via DHCPv6

B4 devices need to know the AFTR's address. Configure your DHCPv6 server to send option 64 (AFTR-Name):

```bash
# ISC DHCP server configuration for AFTR-Name option
# /etc/dhcp/dhcpd6.conf
cat >> /etc/dhcp/dhcpd6.conf << 'EOF'
option aftr-name code 64 = text;
option aftr-name "aftr.example.isp.net";

subnet6 2001:db8:subscriber::/48 {
    range6 2001:db8:subscriber::100 2001:db8:subscriber::ffff;
    option aftr-name "aftr.example.isp.net";
}
EOF
```

## Verifying AFTR Operation

```bash
# Check tunnel interface is up
ip link show aftr0
ip addr show aftr0

# Monitor encapsulated traffic arriving from B4 devices
tcpdump -i eth0 -n 'ip6 proto 4'

# Check NAT44 translation table
conntrack -L -n | head -20

# Test from a B4 device
# On B4: ping -4 8.8.8.8
# On AFTR: watch conntrack -L -p icmp
```

## NAT Logging for Abuse Investigation

At ISP scale, NAT logging is legally required in many jurisdictions:

```bash
# Log NAT translations with conntrack
conntrack -E -p tcp --event-mask NEW | logger -t nat64

# Or use iptables LOG target
iptables -t nat -A POSTROUTING -j LOG --log-prefix "DS-LITE-NAT: "
```

## Summary

Configuring a DS-Lite AFTR on Linux involves creating an `ip4ip6` tunnel interface to accept Softwire connections from B4 devices, configuring iptables NAT44 to translate subscriber IPv4 addresses to the public pool, enabling IP forwarding, and setting proper MTU (1460 bytes) to account for IPv6 tunnel overhead. For production ISP scale, consider dedicated AFTR software like Snabb lwAFTR.
