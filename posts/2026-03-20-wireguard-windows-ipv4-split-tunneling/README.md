# How to Set Up WireGuard on Windows with IPv4 Split Tunneling

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: WireGuard, Windows, VPN, IPv4, Split Tunneling, Networking

Description: Install the WireGuard Windows client and configure an IPv4 split tunnel configuration file to route only selected traffic through the VPN.

WireGuard has a polished GUI client for Windows that makes VPN setup straightforward. Split tunneling lets you route only specific IPv4 subnets through the tunnel while keeping normal internet traffic on the direct connection.

## Step 1: Install WireGuard for Windows

Download the official installer from [wireguard.com/install](https://www.wireguard.com/install/) and run it. The installer adds a system tray icon and a GUI management application.

## Step 2: Generate a Key Pair

Open the WireGuard app, click **Add Tunnel** → **Add empty tunnel**. The application automatically generates a private/public key pair for you. Copy the public key to share with your server administrator.

Alternatively, generate keys from PowerShell if you have WireGuard tools installed:

```powershell
# Generate private key

wg genkey | Tee-Object -FilePath client_private.key | wg pubkey | Out-File client_public.key

# Display keys
Get-Content client_private.key
Get-Content client_public.key
```

## Step 3: Create the Client Configuration

Paste or type the configuration in the WireGuard GUI editor, or create a `.conf` file and import it:

```ini
# windows-client.conf

[Interface]
# Client's private key (generated above)
PrivateKey = <CLIENT_PRIVATE_KEY>

# IPv4 address assigned to this client in the VPN
Address = 10.0.0.5/24

# Optional: use VPN server for DNS
DNS = 10.0.0.1

[Peer]
# Server's public key
PublicKey = <SERVER_PUBLIC_KEY>

# Server's public IP and WireGuard port
Endpoint = 203.0.113.1:51820

# Split tunnel: only these subnets go through the VPN
# Corporate LAN and VPN subnet
AllowedIPs = 10.0.0.0/24, 192.168.100.0/24

# Keep the tunnel alive through NAT
PersistentKeepalive = 25
```

## Step 4: Import and Connect

1. In the WireGuard app, click **Import tunnel(s) from file** and select your `.conf` file.
2. Click **Activate** to connect.
3. The tunnel status changes to "Active" and shows handshake and data transfer statistics.

## Verifying Split Tunneling

Open PowerShell and check that routes were added correctly:

```powershell
# View the routing table and look for routes pointing to the WireGuard interface
route print -4

# Test VPN connectivity
ping 10.0.0.1

# Test that internet traffic bypasses the VPN
# This should return your real public IP, not the VPN server's
Invoke-RestMethod https://ifconfig.me
```

## Toggling Between Split and Full Tunnel

To switch to full tunnel, edit the configuration and change:

```ini
# Full tunnel - all IPv4 traffic through VPN
AllowedIPs = 0.0.0.0/0

# Split tunnel - only specific subnets
AllowedIPs = 10.0.0.0/24, 192.168.100.0/24
```

Deactivate and reactivate the tunnel after editing.

## Troubleshooting on Windows

- Check Windows Firewall is not blocking UDP port 51820 outbound.
- Run `wg show` in an elevated Command Prompt to see peer status.
- Use the WireGuard Log Viewer (accessible from the app menu) for detailed diagnostics.
