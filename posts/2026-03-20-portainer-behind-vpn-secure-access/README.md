# How to Run Portainer Behind a VPN for Secure Access

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, VPN, Security, WireGuard, Network Security

Description: Learn how to restrict Portainer access to VPN-connected users only using WireGuard or other VPN solutions.

## Why Run Portainer Behind a VPN?

Portainer controls your entire container infrastructure. Exposing it to the public internet, even with strong authentication, creates unnecessary risk from:

- Brute-force login attacks.
- Zero-day vulnerabilities in Portainer itself.
- Credential stuffing attacks.

Running Portainer on a private network and requiring VPN access eliminates the attack surface almost entirely.

## Option 1: WireGuard VPN (Recommended)

WireGuard is a modern, fast VPN. Deploy it on the same server as Portainer, or on a separate jump host.

### Install WireGuard

```bash
# Install WireGuard on Ubuntu

apt-get update && apt-get install -y wireguard

# Generate server keys
wg genkey | tee /etc/wireguard/server-private.key | wg pubkey > /etc/wireguard/server-public.key

# Generate client keys (on the client machine)
wg genkey | tee client-private.key | wg pubkey > client-public.key
```

### WireGuard Server Configuration

```ini
# /etc/wireguard/wg0.conf
[Interface]
# Server's WireGuard IP
Address = 10.200.0.1/24
ListenPort = 51820
PrivateKey = <server-private-key>

# Enable routing for connected clients
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

# Client 1 (Admin)
[Peer]
PublicKey = <client-public-key>
AllowedIPs = 10.200.0.2/32
```

```bash
# Start WireGuard
systemctl enable wg-quick@wg0
systemctl start wg-quick@wg0
```

### WireGuard Client Configuration

```ini
# client-wg0.conf
[Interface]
PrivateKey = <client-private-key>
Address = 10.200.0.2/24
DNS = 1.1.1.1

[Peer]
PublicKey = <server-public-key>
Endpoint = portainer-server.mycompany.com:51820
AllowedIPs = 10.200.0.0/24  # Only route VPN traffic through the tunnel
PersistentKeepalive = 25
```

### Bind Portainer to VPN Interface Only

```bash
# Run Portainer only accessible via VPN IP
docker run -d \
  -p 10.200.0.1:9443:9443 \  # Only bind to WireGuard IP
  -p 10.200.0.1:8000:8000 \  # Edge agent port
  --name portainer \
  portainer/portainer-ce:latest
```

## Option 2: Tailscale (Simpler Setup)

Tailscale is a managed WireGuard-based VPN with zero-configuration networking:

```bash
# Install Tailscale on the Portainer server
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up

# Get the Tailscale IP
tailscale ip -4
# Example: 100.64.1.5

# Run Portainer bound to the Tailscale IP
docker run -d \
  -p 100.64.1.5:9443:9443 \
  --name portainer \
  portainer/portainer-ce:latest
```

Users connect via the Tailscale client on their devices and access Portainer at `https://100.64.1.5:9443`.

## Firewall Configuration

Ensure the Portainer port is only accessible from the VPN subnet:

```bash
# UFW: Allow Portainer only from VPN subnet
ufw allow from 10.200.0.0/24 to any port 9443 proto tcp
ufw deny 9443
```

## Conclusion

Running Portainer behind a VPN is the most effective security measure you can take. It removes Portainer from public internet exposure entirely, making it accessible only to authenticated VPN users. WireGuard and Tailscale both provide excellent, easy-to-configure options.
