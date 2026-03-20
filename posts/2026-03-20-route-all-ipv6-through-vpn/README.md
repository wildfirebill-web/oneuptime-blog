# How to Route All IPv6 Traffic Through a VPN Tunnel

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, VPN, Routing, Full Tunnel, Network Security, Privacy

Description: A guide to configuring various VPN clients to route all IPv6 traffic through the tunnel, preventing IPv6 from bypassing VPN protection.

Routing all IPv6 traffic through a VPN requires specific configuration for each VPN protocol. Without explicit configuration, many VPN setups only route IPv4 traffic through the tunnel, leaving IPv6 unprotected. This guide covers per-VPN configuration for full IPv6 routing.

## Why All IPv6 Must Be Routed Through VPN

A split configuration where IPv4 goes through VPN but IPv6 goes direct exposes:
- Your real IPv6 address to every site you visit
- IPv6 traffic to your ISP's monitoring
- Potentially different geographic routing for IPv6

## WireGuard: Route All IPv6

```ini
# /etc/wireguard/wg0.conf

[Interface]
Address = 10.0.0.2/32
Address = fd00:wg::2/128
PrivateKey = <private-key>
DNS = 8.8.8.8, 2001:4860:4860::8888

[Peer]
PublicKey = <server-public-key>
Endpoint = vpn.example.com:51820

# Include ::/0 to route ALL IPv6
AllowedIPs = 0.0.0.0/0, ::/0

PersistentKeepalive = 25
```

```bash
# Verify all IPv6 routes through wg0
ip -6 route show | head -5
# Expected: default dev wg0 metric 1
```

## OpenVPN: Route All IPv6

```ini
# /etc/openvpn/client.conf

client
dev tun
tun-ipv6

remote vpn.example.com 1194 udp6

ca ca.crt
cert client.crt
key client.key

# Route ALL IPv6 through tunnel
route-ipv6 ::/0

pull
```

Server must push IPv6 routes:
```ini
# Server config:
push "route-ipv6 ::/0"
server-ipv6 fd00:vpn::/64
```

## IPsec (strongSwan swanctl): Route All IPv6

```conf
# /etc/swanctl/swanctl.conf

connections {
    full-tunnel {
        version = 2
        remote_addrs = vpn.example.com

        local {
            auth = eap-mschapv2
        }

        remote {
            auth = pubkey
            certs = server-cert.pem
        }

        children {
            all-traffic {
                # Route all IPv6 through tunnel
                local_ts = ::/0
                remote_ts = ::/0
                mode = tunnel
            }
        }

        pools = vpn-ipv6
    }
}

pools {
    vpn-ipv6 {
        addrs = fd00:ipsec::/64
    }
}
```

## Verifying Full IPv6 Tunnel

```bash
# Verify IPv6 default route points to VPN
ip -6 route show default

# For WireGuard:
# default dev wg0 metric 51820

# For OpenVPN:
# default dev tun0 metric 100

# Test that IPv6 traffic exits at VPN server IP
curl -6 https://ifconfig.co
# Should return VPN server's IPv6 address

# Confirm no IPv6 leaks
ping6 -c 2 2001:4860:4860::8888   # Should work (through VPN)
traceroute6 2001:4860:4860::8888  # Should show VPN server's address first
```

## Kill Switch for IPv6

If the VPN disconnects, block all IPv6 traffic:

```bash
# Create kill switch rules (run before connecting to VPN)
sudo ip6tables -A OUTPUT -o lo -j ACCEPT
sudo ip6tables -A OUTPUT -o wg0 -j ACCEPT    # Allow through VPN
sudo ip6tables -A OUTPUT -j DROP              # Block all other IPv6

# Or for OpenVPN:
sudo ip6tables -A OUTPUT -o tun0 -j ACCEPT
sudo ip6tables -A OUTPUT -j DROP
```

## NetworkManager Configuration

For GUI-managed VPNs:

```bash
# For WireGuard in NetworkManager
nmcli connection modify "WireGuard VPN" \
  ipv6.method auto \
  ipv6.routes "::/0"

# For OpenVPN in NetworkManager
# Enable "Use this connection only for resources on its network" = OFF
# This ensures all traffic including IPv6 routes through VPN
```

## Testing Full IPv6 Routing

```bash
#!/bin/bash
# test-full-ipv6-vpn.sh

VPN_SERVER_IPV6="<your-vpn-server-ipv6>"

echo "Testing IPv6 routing through VPN..."

# Get current exit IPv6
MY_IPV6=$(curl -s -6 https://ifconfig.co 2>/dev/null)

if [ "$MY_IPV6" = "$VPN_SERVER_IPV6" ]; then
    echo "PASS: IPv6 exits at VPN server ($MY_IPV6)"
elif [ -z "$MY_IPV6" ]; then
    echo "INFO: No IPv6 connectivity (may be blocked)"
else
    echo "FAIL: IPv6 exits at $MY_IPV6 (not VPN server)"
fi
```

Full IPv6 tunnel routing is essential for any deployment where consistent security policy for all traffic is required, whether for privacy, compliance, or enterprise security requirements.
