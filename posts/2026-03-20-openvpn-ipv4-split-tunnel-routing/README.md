# How to Configure OpenVPN Client Routing for IPv4 Split Tunnel

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenVPN, VPN, IPv4, Split Tunneling, Routing, Networking

Description: Configure OpenVPN to only route specific IPv4 subnets through the VPN tunnel while leaving direct internet traffic unaffected.

By default, OpenVPN in full-tunnel mode routes all traffic through the VPN. Split tunneling sends only designated IPv4 subnets through the tunnel, improving performance and preserving bandwidth for internet-bound traffic.

## Server-Side: Avoid Pushing the Default Route

In your OpenVPN server config, remove or comment out the default route push:

```conf
# /etc/openvpn/server/server.conf

# Remove or comment out this line for split tunneling

# push "redirect-gateway def1 bypass-dhcp"

# Instead, push only specific routes to clients
push "route 192.168.10.0 255.255.255.0"
push "route 10.8.0.0 255.255.255.0"
```

## Client-Side: Override Server Push

If you control only the client, you can prevent the server's pushed routes from taking effect using `route-nopull` and adding routes manually:

```conf
# client.ovpn

client
dev tun
proto udp
remote 203.0.113.1 1194
resolv-retry infinite
nobind

# Ignore routes pushed by the server
route-nopull

# Manually add only the routes you want through the VPN
route 10.8.0.0 255.255.255.0
route 192.168.10.0 255.255.255.0

# Include certificates below
<ca>
...
</ca>
```

## Client-Side: Use a Script for Dynamic Route Management

For more complex scenarios, use an `up` script to add routes after the tunnel comes up:

```bash
# /etc/openvpn/add-routes.sh

#!/bin/bash
# Add specific routes through the VPN tunnel
ip route add 192.168.10.0/24 via $route_vpn_gateway
ip route add 172.16.0.0/16 via $route_vpn_gateway
```

Reference it in the client config:

```conf
script-security 2
up /etc/openvpn/add-routes.sh
```

## Verifying Split Tunnel Routes

```bash
# After connecting, check the routing table
ip route show

# Traffic to VPN subnets should use the tun interface
ip route get 192.168.10.5
# Expected: 192.168.10.5 via 10.8.0.1 dev tun0

# Internet traffic should use the original default gateway
ip route get 8.8.8.8
# Expected: 8.8.8.8 via 192.168.1.1 dev eth0
```

## Handling DNS in Split Tunnel Mode

Without pushing a DNS server, clients may still send all DNS queries through their normal interface. If internal hostnames need to resolve over VPN, configure the DNS client to use VPN DNS for specific domains:

```conf
# Push split DNS to OpenVPN clients
push "dhcp-option DNS 10.8.0.1"
push "dhcp-option DOMAIN corp.example.com"
```

Split tunneling reduces VPN load and latency for general internet use while keeping internal corporate traffic secure through the encrypted tunnel.
