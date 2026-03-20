# How to Configure WireGuard Full Tunnel Routing for All IPv4 Traffic

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: WireGuard, VPN, IPv4, Full Tunnel, Routing, Linux

Description: Configure WireGuard to route all IPv4 traffic through the VPN tunnel, including internet-bound traffic, using AllowedIPs and policy routing.

Full tunnel mode sends every IPv4 packet from the client through the WireGuard interface. This is useful for privacy, corporate policy compliance, or ensuring all traffic exits through a trusted egress point.

## The AllowedIPs Setting for Full Tunnel

The key to full tunnel routing is a single `AllowedIPs` entry on the client peer:

```ini
# This tells WireGuard to route ALL IPv4 traffic through the tunnel
AllowedIPs = 0.0.0.0/0
```

## Client Configuration for Full Tunnel

```ini
# /etc/wireguard/wg0.conf (client - full tunnel)

[Interface]
PrivateKey = <CLIENT_PRIVATE_KEY>
Address = 10.0.0.2/24
# Use the VPN server's DNS to prevent DNS leaks
DNS = 10.0.0.1

[Peer]
PublicKey = <SERVER_PUBLIC_KEY>
# Server's public IP MUST NOT be routed through the tunnel (routing loop prevention)
Endpoint = 203.0.113.1:51820
# Route everything through the VPN
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

## Why wg-quick Handles the Default Route Specially

When `AllowedIPs = 0.0.0.0/0`, `wg-quick` uses a special trick to avoid a routing loop. It adds the WireGuard server's IP as a host route via the original default gateway, then sets a fwmark-based policy route to direct all other traffic through `wg0`.

```bash
# wg-quick automatically performs these steps:
# 1. Add a host route for the VPN server IP via the real gateway
ip route add 203.0.113.1/32 via 192.168.1.1

# 2. Add a policy routing rule for WireGuard-marked packets
ip rule add not fwmark 51820 table 51820
ip route add default dev wg0 table 51820
```

You don't need to run these manually — `wg-quick up wg0` handles them automatically.

## Server-Side NAT for Full Tunnel

The server must masquerade outbound internet traffic for VPN clients:

```ini
# On the server's wg0.conf
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
```

Replace `eth0` with your server's actual internet-facing interface.

## Verifying Full Tunnel is Active

```bash
# Bring up the tunnel
sudo wg-quick up wg0

# Check your public IP - it should now be the VPN server's IP
curl https://ifconfig.me

# Verify the default route uses wg0
ip route get 8.8.8.8
# Expected: 8.8.8.8 dev wg0 ...
```

## Preventing DNS Leaks

Set the `DNS` field in the `[Interface]` section to a DNS server reachable over the VPN, or use `1.1.1.1` / `8.8.8.8` if your server forwards DNS. This prevents DNS queries from leaking outside the tunnel.
