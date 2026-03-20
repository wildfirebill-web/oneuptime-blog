# How to Troubleshoot IPv6 Link-Local Address Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Link-Local, Troubleshooting, Fe80, Scope ID, NDP

Description: Diagnose and fix IPv6 link-local address issues including missing fe80 addresses, scope ID requirements, and routing problems with fe80::/10 prefixes.

## Introduction

IPv6 link-local addresses (`fe80::/10`) are automatically assigned to every IPv6-enabled interface and are required for NDP, router discovery, and same-link communication. Unlike global addresses, link-local addresses are never forwarded by routers and require a scope ID (interface name) when used outside the kernel. Issues with link-local addresses can prevent SLAAC, NDP, and default gateway discovery from working.

## Step 1: Verify Link-Local Address Exists

```bash
# Show all link-local addresses

ip -6 addr show scope link

# Show link-local for specific interface
ip -6 addr show dev eth0 | grep "scope link"

# Expected output:
# inet6 fe80::1234:5678:abcd:ef01/64 scope link

# If no link-local address:
# → IPv6 is disabled (disable_ipv6=1)
# → Interface is down
# → Kernel issue
```

## Step 2: Generate Link-Local Address

```bash
# If link-local is missing, IPv6 may be disabled
cat /proc/sys/net/ipv6/conf/eth0/disable_ipv6

# Enable IPv6
sudo sysctl -w net.ipv6.conf.eth0.disable_ipv6=0

# Bring interface down and up to trigger address generation
sudo ip link set eth0 down && sudo ip link set eth0 up

# Or manually add a link-local address
sudo ip -6 addr add fe80::1/64 dev eth0 scope link

# Verify
ip -6 addr show dev eth0 | grep "scope link"
```

## Step 3: Using Scope IDs Correctly

Link-local addresses require a scope ID when used in commands or URLs:

```bash
# Ping a link-local address - must specify interface
ping6 fe80::1%eth0
# The %eth0 is the scope ID

# Without scope ID, ping6 fails:
ping6 fe80::1  # Error: network is unreachable

# SSH to a link-local address
ssh user@fe80::1%eth0

# curl with link-local (URL-encode the %)
curl http://[fe80::1%25eth0]/  # %25 = URL-encoded %

# Check neighbor at link-local
ndisc6 fe80::1 eth0

# Traceroute to link-local
traceroute6 fe80::1%eth0
```

## Step 4: Link-Local as Default Gateway

```bash
# Routers use link-local addresses as default gateway next-hops
ip -6 route show default
# Example: default via fe80::1 dev eth0 proto ra

# The gateway fe80::1 must be on the same link as eth0
# Verify gateway is reachable
ping6 -I eth0 fe80::1  # Must specify interface

# If gateway is unreachable:
# → Check physical connectivity
# → Check if router is configured
# → Check NDP cache for gateway
ip -6 neigh show dev eth0 | grep fe80
```

## Step 5: Duplicate Link-Local Addresses

```bash
# Link-local DAD failure means another device has the same fe80:: address
# This is rare since link-local is derived from MAC address

# Check for DAD failure
ip -6 addr show dev eth0 | grep "tentative\|dadfailed"

# If link-local is dadfailed:
# → Another device has the same MAC-derived address
# → Use a custom link-local address
sudo ip -6 addr del fe80::DADFAILED/64 dev eth0
sudo ip -6 addr add fe80::abcd:1234:5678:9abc/64 dev eth0 scope link

# Ensure no conflicts
ndisc6 fe80::abcd:1234:5678:9abc eth0
# Should timeout (no other device has this address)
```

## Step 6: Troubleshoot Scope ID in Applications

```python
# Python: handling scope IDs
import socket

# getaddrinfo returns scope ID as part of the address tuple
info = socket.getaddrinfo("fe80::1%eth0", 80,
                           socket.AF_INET6,
                           socket.SOCK_STREAM)
for res in info:
    af, socktype, proto, canonname, sockaddr = res
    print(f"Address: {sockaddr}")
    # sockaddr = ('fe80::1', 80, 0, interface_index)
    # The 4th element is the scope ID (interface index)

# Get interface index
scope_id = socket.if_nametoindex("eth0")
sockaddr = ("fe80::1", 80, 0, scope_id)

sock = socket.socket(socket.AF_INET6, socket.SOCK_STREAM)
sock.connect(sockaddr)
```

## Diagnostic Script

```bash
#!/bin/bash
# diagnose-link-local.sh

echo "=== IPv6 Link-Local Diagnostics ==="

echo ""
echo "Link-local addresses:"
ip -6 addr show scope link | grep "inet6\|^[0-9]"

echo ""
echo "Default gateway via link-local:"
ip -6 route show default | grep "fe80"

echo ""
echo "Link-local neighbors:"
ip -6 neigh show | grep "^fe80"

echo ""
echo "Reachability of link-local gateways:"
while IFS= read -r line; do
    gw=$(echo "$line" | awk '/fe80/{print $3}')
    dev=$(echo "$line" | awk '/fe80/{print $5}')
    if [ -n "$gw" ] && [ -n "$dev" ]; then
        ping6 -c 1 -W 2 -I "$dev" "$gw" &>/dev/null && \
            echo "  [OK] $gw via $dev" || \
            echo "  [FAIL] $gw via $dev"
    fi
done < <(ip -6 route show default)
```

## Conclusion

IPv6 link-local addresses are fundamental to IPv6 operation - without them, NDP and router discovery cannot function. Verify their existence with `ip -6 addr show scope link`. When using link-local addresses in commands or applications, always include the scope ID (`%eth0`) since link-local addresses are not globally unique. The most common link-local issue is forgetting the scope ID, which causes "network is unreachable" errors even when the address exists and the device is reachable.
