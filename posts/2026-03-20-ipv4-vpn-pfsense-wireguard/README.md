# How to Configure IPv4 VPN on pfSense with WireGuard

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: WireGuard, pfSense, VPN, IPv4, Firewall, Networking

Description: Set up a WireGuard VPN tunnel on pfSense for IPv4 remote access, including interface creation, peer configuration, and firewall rules.

WireGuard is natively supported in pfSense 2.5.2 and later via a built-in package. It provides a high-performance VPN option for pfSense-based networks.

## Step 1: Install WireGuard Package (if needed)

In pfSense 2.5.2+, WireGuard is built-in. On older versions:

1. Navigate to **System → Package Manager → Available Packages**
2. Search for `WireGuard` and install it

## Step 2: Create a WireGuard Tunnel

1. Navigate to **VPN → WireGuard → Tunnels**
2. Click **+ Add Tunnel**
3. Configure the tunnel:
   - **Description:** My WireGuard VPN
   - **Listen Port:** 51820
   - Click **Generate** to create a key pair
   - **Interface Address:** `10.0.0.1/24` (VPN subnet for this server)
4. Click **Save Tunnel**

## Step 3: Add Peers

1. Click the **Peers** tab, then **+ Add Peer**
2. Fill in peer details:
   - **Tunnel:** Select your tunnel
   - **Description:** Remote client name
   - **Public Key:** Paste the client's public key
   - **Allowed IPs:** `10.0.0.2/32` (client's VPN IP)
   - **Endpoint:** Leave blank (client initiates)
   - **Keepalive:** 25
3. Click **Save Peer**

## Step 4: Assign the WireGuard Interface

1. Go to **Interfaces → Assignments**
2. Find the `wg0` interface in the available network ports dropdown
3. Click **+ Add** and save
4. Navigate to the newly created interface (e.g., **OPT1**)
5. Enable it, set a description like `WireGuard`, and save

## Step 5: Configure Firewall Rules

Navigate to **Firewall → Rules**.

**WAN Rules** - Allow incoming WireGuard traffic:
- Action: Pass
- Interface: WAN
- Protocol: UDP
- Destination Port: 51820
- Description: Allow WireGuard VPN

**WireGuard Interface Rules** - Allow VPN client traffic:
- Action: Pass
- Interface: WireGuard
- Source: WireGuard subnet (`10.0.0.0/24`)
- Destination: Any
- Description: Allow VPN clients

## Step 6: Configure NAT for Internet Access

1. Go to **Firewall → NAT → Outbound**
2. Switch to **Hybrid** or **Manual** outbound NAT mode
3. Add a rule to masquerade WireGuard traffic:
   - Interface: WAN
   - Source: `10.0.0.0/24`
   - Translation Address: Interface Address

## Step 7: Generate Client Config

```ini
# Client wg0.conf

[Interface]
PrivateKey = <CLIENT_PRIVATE_KEY>
Address = 10.0.0.2/24
DNS = 8.8.8.8

[Peer]
PublicKey = <PFSENSE_WIREGUARD_PUBLIC_KEY>
Endpoint = <PFSENSE_PUBLIC_IP>:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

After applying, clients connecting to the pfSense WireGuard server will receive a VPN IP in the `10.0.0.0/24` range and can access the internal network.
