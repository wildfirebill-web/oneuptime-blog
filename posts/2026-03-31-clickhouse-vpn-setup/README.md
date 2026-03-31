# How to Set Up ClickHouse with a VPN

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, VPN, Security, WireGuard, Networking

Description: Learn how to set up ClickHouse access over a VPN using WireGuard to secure connections without exposing the database to the public internet.

---

## Why Use a VPN with ClickHouse?

Exposing ClickHouse directly to the internet creates significant security risk. Even with authentication, the database surface area is too large for public exposure. A VPN confines access to authenticated VPN users only, adding a network-layer control that complements ClickHouse's own user authentication.

## Architecture

```text
Developer Laptop
    |
  WireGuard VPN
    |
  VPN Gateway (10.8.0.1)
    |
  Private Network
    |
  ClickHouse (10.0.0.5:8123, :9000)
```

ClickHouse listens only on private network interfaces. Clients must be connected to the VPN to reach it.

## Setting Up WireGuard (Server Side)

```bash
# Install WireGuard
sudo apt install wireguard

# Generate server keys
wg genkey | tee /etc/wireguard/server_private.key | wg pubkey > /etc/wireguard/server_public.key
```

```ini
# /etc/wireguard/wg0.conf (server)
[Interface]
Address = 10.8.0.1/24
PrivateKey = <server_private_key>
ListenPort = 51820
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = <client_public_key>
AllowedIPs = 10.8.0.2/32
```

## Setting Up WireGuard (Client Side)

```ini
# /etc/wireguard/wg0.conf (client)
[Interface]
Address = 10.8.0.2/24
PrivateKey = <client_private_key>
DNS = 10.8.0.1

[Peer]
PublicKey = <server_public_key>
Endpoint = vpn-server-public-ip:51820
AllowedIPs = 10.0.0.0/8
PersistentKeepalive = 25
```

## Bind ClickHouse to Private Interface Only

In `config.xml`, restrict ClickHouse to the private network:

```xml
<listen_host>10.0.0.5</listen_host>
```

After this change, ClickHouse is not reachable without VPN access.

## Firewall Rules

```bash
# Allow ClickHouse only from VPN subnet
sudo ufw allow from 10.8.0.0/24 to any port 8123
sudo ufw allow from 10.8.0.0/24 to any port 9000
sudo ufw deny 8123
sudo ufw deny 9000
```

## Connecting to ClickHouse Over VPN

```bash
# Start VPN
sudo wg-quick up wg0

# Connect to ClickHouse
clickhouse-client --host 10.0.0.5 --query "SELECT 1"
curl "http://10.0.0.5:8123/?query=SELECT+1"
```

## Monitoring VPN Connections

```bash
# View active WireGuard peers
sudo wg show

# Output shows handshake time and data transfer per peer
```

## Summary

Securing ClickHouse with WireGuard VPN keeps the database off the public internet while providing encrypted access for authorized users. Bind ClickHouse to private network interfaces, use firewall rules to allow only VPN subnet traffic, and configure `PersistentKeepalive` on clients to prevent idle VPN connections from dropping during long queries.
