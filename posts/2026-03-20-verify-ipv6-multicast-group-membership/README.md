# How to Verify IPv6 Multicast Group Membership

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Multicast, MLD, Network Verification, Linux

Description: A guide to verifying IPv6 multicast group membership on Linux hosts, switches, and routers using command-line tools and packet captures.

## Viewing Multicast Groups on Linux Hosts

```bash
# List all IPv6 multicast groups on all interfaces
ip -6 maddr show

# List groups on a specific interface
ip -6 maddr show dev eth0

# Example output:
# 3: eth0
#     inet6 ff02::1 users 3 flags permanent
#     inet6 ff02::1:ff00:1 users 1 flags permanent
#     inet6 ff02::2 users 1 flags permanent      (if forwarding enabled)
#     inet6 ff3e::db8:stream users 1 flags dynamic  (application-joined)
```

## Understanding the ip -6 maddr Output

The output format is:
```
<index>: <interface>
    inet6 <multicast-group> users <count> flags <flags>
```

**flags**:
- `permanent`: System-joined (all nodes, solicited-node) — cannot be removed
- `dynamic`: Application-joined (via socket) — removed when socket closes
- `static`: Manually added with `ip -6 maddr add`

**users**: Number of sockets/processes that have joined this group

## Checking Which Process Joined a Group

```bash
# Find which socket has joined a specific multicast group
# Use ss to show sockets with multicast membership
ss -6 -unlp | head

# Or use netstat
netstat -6 -g
# Shows all multicast group memberships by interface

# More detailed: check /proc for socket info
# For each socket, check if it joined multicast
cat /proc/net/ipv6 2>/dev/null  # socket table

# Python: list all multicast memberships via netlink
python3 -c "
import socket
# Query multicast groups from kernel
s = socket.socket(socket.AF_NETLINK, socket.SOCK_RAW, socket.NETLINK_ROUTE)
# This requires additional library support (pyroute2)
print('Use: ip -6 maddr show')
"
```

## Adding and Removing Groups Manually

```bash
# Add a static multicast group membership (for testing)
ip -6 maddr add ff3e::db8:test dev eth0

# Verify it was added
ip -6 maddr show dev eth0 | grep ff3e

# Remove the static group membership
ip -6 maddr delete ff3e::db8:test dev eth0

# Or use the add flag with 'static' type
ip -6 maddr add ff3e::db8:test dev eth0 autojoin
```

## Verifying MLD Reports are Sent

When a group is joined, the kernel sends an MLD report. Verify this:

```bash
# Start capturing MLD reports before joining
tcpdump -i eth0 -n 'icmp6 and ip6[40] == 143' &

# Join a multicast group (triggers MLD report)
ip -6 maddr add ff3e::db8:test dev eth0

# Should see an MLDv2 Report in the capture
fg; # Ctrl+C to stop capture
```

## Checking Group Membership from the Router Perspective

On the last-hop router (must have PIM enabled), verify it knows about group members:

**FRR (Linux)**:
```bash
# Check MLD group table on the router
vtysh -c "show ipv6 mld groups"

# Expected output:
# Interface  Group            Source    Last Reporter   Uptime Expires
# eth0       ff3e::db8:stream *         2001:db8::host  00:05  00:03:55
```

**Cisco IOS**:
```cisco
show ipv6 mld groups
show ipv6 mld groups detail
```

**Juniper JunOS**:
```juniper
run show mld group inet6
run show mld group inet6 detail
```

## Testing Multicast Delivery End-to-End

```bash
# On receiver: join the group and listen
python3 -c "
import socket, struct
sock = socket.socket(socket.AF_INET6, socket.SOCK_DGRAM)
sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
sock.bind(('', 5000))
group = socket.inet_pton(socket.AF_INET6, 'ff3e::db8:test')
ifidx = struct.pack('I', socket.if_nametoindex('eth0'))
sock.setsockopt(socket.IPPROTO_IPV6, socket.IPV6_JOIN_GROUP, group + ifidx)
print('Listening on ff3e::db8:test port 5000')
while True:
    data, addr = sock.recvfrom(1024)
    print(f'Received: {data} from {addr[0]}')
"

# On source (different host): send test packets
python3 -c "
import socket, time
sock = socket.socket(socket.AF_INET6, socket.SOCK_DGRAM)
sock.setsockopt(socket.IPPROTO_IPV6, socket.IPV6_MULTICAST_HOPS, 32)
for i in range(5):
    sock.sendto(f'test packet {i}'.encode(), ('ff3e::db8:test', 5000))
    print(f'Sent packet {i}')
    time.sleep(1)
"
```

## Checking Multicast Group Membership on Network Switches

On managed switches with MLD snooping enabled, check the multicast database:

**Linux bridge**:
```bash
# Check bridge multicast database
bridge mdb show

# Filter for specific group
bridge mdb show | grep 'ff3e'
```

**Cisco switch**:
```cisco
show ipv6 mld snooping groups vlan 100
show ipv6 mld snooping groups vlan 100 detail
```

## Summary

Verify IPv6 multicast group membership with `ip -6 maddr show` on Linux hosts, `show ipv6 mld groups` on Cisco/FRR routers, and `bridge mdb show` on Linux bridges. When a process joins a group, the kernel automatically sends an MLD report — capture this with `tcpdump -n 'icmp6 and ip6[40] == 143'`. For end-to-end verification, use a sender/receiver test to confirm actual data delivery.
