# How to Configure WireGuard Client Peers with IPv4 Addresses

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: WireGuard, VPN, IPv4, Networking, Linux, Peer Configuration

Description: Learn how to generate client key pairs and add WireGuard peer entries on both the server and client side for IPv4 connectivity.

Once your WireGuard server is running, you need to configure peers (clients) so they can connect. Each peer has its own private/public key pair and a unique IPv4 address within the VPN subnet.

## Step 1: Generate Client Keys

Run this on the client machine (or generate server-side and distribute securely):

```bash
# Generate a private key and derive the public key from it
wg genkey | tee client_private.key | wg pubkey > client_public.key

# Protect the private key
chmod 600 client_private.key

# Display the keys
cat client_private.key
cat client_public.key
```

## Step 2: Add the Peer to the Server

On the server, add a `[Peer]` section to `/etc/wireguard/wg0.conf`, or use the `wg set` command for live changes.

Using a config file edit (requires interface restart):

```ini
# Append to /etc/wireguard/wg0.conf

[Peer]
# The client's public key
PublicKey = <CLIENT_PUBLIC_KEY>

# The IPv4 address assigned to this client inside the VPN
AllowedIPs = 10.0.0.2/32
```

Using `wg set` for a live update (no restart needed):

```bash
# Add peer without restarting WireGuard
sudo wg set wg0 peer <CLIENT_PUBLIC_KEY> allowed-ips 10.0.0.2/32
```

## Step 3: Create the Client Configuration

On the client machine, create `/etc/wireguard/wg0.conf`:

```ini
# /etc/wireguard/wg0.conf (client)

[Interface]
# The client's private key
PrivateKey = <CLIENT_PRIVATE_KEY>

# The IPv4 address assigned to this client
Address = 10.0.0.2/24

# Optional: specify DNS server to use over the VPN
DNS = 1.1.1.1

[Peer]
# The server's public key
PublicKey = <SERVER_PUBLIC_KEY>

# The server's public IPv4 address and WireGuard port
Endpoint = 203.0.113.1:51820

# Which traffic to route through the VPN
# Use 0.0.0.0/0 for full tunnel or specific CIDRs for split tunneling
AllowedIPs = 10.0.0.0/24

# Send a keepalive packet every 25 seconds to maintain NAT mappings
PersistentKeepalive = 25
```

## Step 4: Connect from the Client

```bash
# Bring up the WireGuard interface on the client
sudo wg-quick up wg0

# Verify the connection status
sudo wg show

# Test connectivity to the server's VPN IP
ping 10.0.0.1
```

## Verifying Peer Status on the Server

```bash
# Check connected peers, handshake time, and data transfer
sudo wg show wg0
```

A successful peer connection shows a recent `latest handshake` timestamp and increasing `transfer` bytes. Each additional client gets a unique `AllowedIPs` address like `10.0.0.3/32`, `10.0.0.4/32`, and so on.
