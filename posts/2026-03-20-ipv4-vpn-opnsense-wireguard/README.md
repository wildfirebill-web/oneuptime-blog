# How to Configure IPv4 VPN on OPNsense with WireGuard

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: WireGuard, OPNsense, VPN, IPv4, Firewall, Networking

Description: Install the WireGuard plugin on OPNsense and configure a WireGuard IPv4 VPN with firewall rules for remote access.

OPNsense supports WireGuard via a community plugin. The configuration workflow is similar to pfSense but uses OPNsense's Instances and Peers terminology.

## Step 1: Install the WireGuard Plugin

1. Go to **System → Firmware → Plugins**
2. Search for `os-wireguard`
3. Click **+** to install the plugin
4. Refresh the page after installation

## Step 2: Create a WireGuard Instance (Server)

1. Navigate to **VPN → WireGuard → Local**
2. Click **+** to add a new instance
3. Configure:
   - **Name:** wg_server
   - **Enabled:** checked
   - **Listen Port:** 51820
   - Click the key icon to **Generate** private/public key pair
   - **Tunnel Address:** `10.0.0.1/24`
   - **DNS Server:** Leave blank or set `8.8.8.8`
4. Save and note the **Public Key** — clients will need this

## Step 3: Add a Peer (Client)

1. Go to **VPN → WireGuard → Peers**
2. Click **+** to add a peer
3. Fill in:
   - **Name:** Client1
   - **Enabled:** checked
   - **Public Key:** Client's public key
   - **Allowed IPs:** `10.0.0.2/32`
   - **Instance:** Select your wg_server instance
4. Save

## Step 4: Assign and Enable the Interface

1. Go to **Interfaces → Assignments**
2. Find `wg_server` in the available ports and assign it (creates `opt1` or similar)
3. Go to **Interfaces → [your new interface]**
4. Enable it, give it a description like `WireGuard`
5. Save and apply

## Step 5: Enable the Service

1. Go to **VPN → WireGuard → General**
2. Check **Enable WireGuard**
3. Save and apply

## Step 6: Create Firewall Rules

**WAN Rules** — Allow WireGuard handshake:
- Go to **Firewall → Rules → WAN**
- Add rule: Protocol UDP, Destination Port 51820, Action Pass

**WireGuard Interface Rules** — Allow client traffic:
- Go to **Firewall → Rules → WireGuard**
- Add rule: Source any, Destination any, Action Pass (or restrict as needed)

## Step 7: Configure Outbound NAT

1. Go to **Firewall → NAT → Outbound**
2. Switch to **Hybrid** mode
3. Add a manual rule:
   - Interface: WAN
   - Source: `10.0.0.0/24`
   - Translation: Interface address

## Client Configuration

```ini
# client.conf

[Interface]
PrivateKey = <CLIENT_PRIVATE_KEY>
Address = 10.0.0.2/24
DNS = 8.8.8.8

[Peer]
PublicKey = <OPNSENSE_WIREGUARD_PUBLIC_KEY>
Endpoint = <OPNSENSE_PUBLIC_IP>:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

After applying, verify the tunnel status under **VPN → WireGuard → Status** — a successful handshake shows the peer's public key with a recent timestamp and data transfer stats.
