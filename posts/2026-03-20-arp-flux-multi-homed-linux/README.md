# How to Understand ARP Flux on Multi-Homed Linux Hosts

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Networking, ARP, Linux, Multi-Homed, IPv4

Description: Learn what ARP flux is, why it causes connectivity issues on multi-homed Linux hosts, and how to fix it with arp_filter and arp_ignore.

## What Is ARP Flux?

ARP flux (also called ARP gratuitous confusion) occurs on a Linux host with **multiple network interfaces** (multi-homed). The kernel may:

1. Reply to ARP requests on the "wrong" interface
2. Use a different source IP in ARP replies than the interface that received the request
3. Advertise IP addresses that belong to another interface

This can cause routing asymmetry and connectivity issues in environments with multiple uplinks or complex routing.

## Why ARP Flux Happens

By default, Linux will respond to ARP requests for any IP configured on any interface, regardless of which interface received the request.

**Example:**

```text
eth0: 192.168.1.10/24
eth1: 10.0.0.10/24

ARP Request on eth1: "Who has 192.168.1.10?"
Linux may reply on eth1 using 192.168.1.10
→ ARP reply comes from the wrong interface
```

This confuses routers and switches that expect consistent IP-interface mappings.

## Checking Current ARP Behavior

```bash
# Check arp_ignore and arp_filter settings for each interface

for iface in /proc/sys/net/ipv4/conf/*/; do
    name=$(basename "$iface")
    ignore=$(cat "${iface}arp_ignore")
    filter=$(cat "${iface}arp_filter")
    echo "Interface: $name  arp_ignore=$ignore  arp_filter=$filter"
done
```

## Fixing ARP Flux: arp_ignore

The `arp_ignore` parameter controls which ARP requests Linux responds to:

| Value | Behavior |
|-------|---------|
| 0 | Reply to any ARP request for any IP on any interface (default, causes flux) |
| 1 | Only reply if the target IP matches the interface that received the request |
| 2 | Only reply if target IP matches interface AND source IP is on the same subnet |
| 3-8 | More restrictive modes (rarely used) |

### Set arp_ignore=1 (Recommended for Multi-Homed Hosts)

```bash
# Per interface
sudo sysctl -w net.ipv4.conf.eth0.arp_ignore=1
sudo sysctl -w net.ipv4.conf.eth1.arp_ignore=1

# Or globally (affects all interfaces)
sudo sysctl -w net.ipv4.conf.all.arp_ignore=1
```

## Fixing ARP Flux: arp_announce

The `arp_announce` parameter controls which source IP is used in ARP requests sent by this host:

| Value | Behavior |
|-------|---------|
| 0 | Use any local IP as source (default, can cause flux) |
| 1 | Avoid using IP addresses not in the target's subnet |
| 2 | Always use the IP of the interface sending the request |

```bash
# Recommended for multi-homed
sudo sysctl -w net.ipv4.conf.all.arp_announce=2
```

## Fixing ARP Flux: arp_filter

```bash
# arp_filter=1: Only answer ARP if the kernel would route the reply through
# the receiving interface (requires policy routing to be configured)
sudo sysctl -w net.ipv4.conf.all.arp_filter=1
```

## Persistent Configuration

```bash
cat >> /etc/sysctl.conf << 'EOF'
net.ipv4.conf.all.arp_ignore = 1
net.ipv4.conf.all.arp_announce = 2
EOF
sudo sysctl -p
```

## When ARP Flux Is a Problem

ARP flux especially affects:

- **Load balancers with multiple NICs** (e.g., LVS/IPVS)
- **Bonded/LACP interfaces** with multiple physical NICs
- **Hosts with management + data NICs**
- **Container hosts** with multiple bridge networks

## Key Takeaways

- ARP flux occurs when Linux responds to ARP requests on the wrong interface.
- `arp_ignore=1` limits replies to only the interface that received the request.
- `arp_announce=2` ensures ARP requests use the source IP of the outgoing interface.
- These settings are essential for multi-homed servers, LVS load balancers, and ECMP routing setups.

**Related Reading:**

- [How to View the ARP Table on Linux](https://oneuptime.com/blog/post/2026-03-20-view-arp-table-linux/view)
- [How to Troubleshoot ARP Resolution Failures](https://oneuptime.com/blog/post/2026-03-20-troubleshoot-arp-resolution/view)
- [How to Configure Policy-Based Routing on Linux](https://oneuptime.com/blog/post/2026-03-20-policy-based-routing-linux/view)
