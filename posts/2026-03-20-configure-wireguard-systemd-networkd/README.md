# How to Configure a WireGuard VPN with systemd-networkd

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, systemd-networkd, WireGuard, VPN, Networking, Configuration

Description: Configure a WireGuard VPN tunnel using systemd-networkd .netdev and .network files for persistent, declarative VPN setup.

## Introduction

systemd-networkd supports WireGuard via `.netdev` files with `Kind=wireguard`. Keys and peer configuration go in the `[WireGuard]` and `[WireGuardPeer]` sections. This is an alternative to the `wg-quick` tool for WireGuard management.

## Generate WireGuard Keys

```bash
# Generate a private key
wg genkey | tee /etc/systemd/network/wg0.key | wg pubkey > /etc/systemd/network/wg0.pub

# Restrict permissions on the private key
chmod 600 /etc/systemd/network/wg0.key

# Display the keys
cat /etc/systemd/network/wg0.key    # private key
cat /etc/systemd/network/wg0.pub    # public key
```

## Create the WireGuard Device (.netdev)

```ini
# /etc/systemd/network/99-wg0.netdev
[NetDev]
Name=wg0
Kind=wireguard

[WireGuard]
ListenPort=51820
PrivateKeyFile=/etc/systemd/network/wg0.key

[WireGuardPeer]
# Replace with the peer's actual public key
PublicKey=<PEER_PUBLIC_KEY>
AllowedIPs=10.0.0.2/32
Endpoint=203.0.113.2:51820
PersistentKeepalive=25
```

## Configure the WireGuard IP (.network)

```ini
# /etc/systemd/network/99-wg0.network
[Match]
Name=wg0

[Network]
Address=10.0.0.1/24
```

## Apply and Verify

```bash
# Restart to create the WireGuard interface
systemctl restart systemd-networkd

# Check WireGuard status
wg show wg0

# Show interface
ip addr show wg0

# Test peer connectivity
ping -c 3 10.0.0.2
```

## Peer Configuration (Remote Host)

On the remote peer:

```ini
# /etc/systemd/network/99-wg0.netdev
[NetDev]
Name=wg0
Kind=wireguard

[WireGuard]
ListenPort=51820
PrivateKeyFile=/etc/systemd/network/wg0.key

[WireGuardPeer]
PublicKey=<SERVER_PUBLIC_KEY>
AllowedIPs=10.0.0.1/32
Endpoint=203.0.113.1:51820
PersistentKeepalive=25
```

```ini
# /etc/systemd/network/99-wg0.network
[Match]
Name=wg0

[Network]
Address=10.0.0.2/24
```

## Full Tunnel (Route All Traffic Through VPN)

```ini
[WireGuardPeer]
PublicKey=<PEER_PUBLIC_KEY>
# Route everything through the VPN
AllowedIPs=0.0.0.0/0
Endpoint=203.0.113.2:51820
PersistentKeepalive=25
```

```ini
# In .network, add the default route
[Route]
Destination=0.0.0.0/0
```

## Conclusion

WireGuard in systemd-networkd is configured through `.netdev` files with `Kind=wireguard`. Store private keys in files referenced by `PrivateKeyFile`. Define peers in `[WireGuardPeer]` sections with their public keys, allowed IPs, and endpoint. Verify with `wg show` and `networkctl status wg0`.
