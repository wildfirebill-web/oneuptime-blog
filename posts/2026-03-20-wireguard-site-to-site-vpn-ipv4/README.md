# How to Set Up WireGuard Site-to-Site VPN Between Two IPv4 Networks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: WireGuard, VPN, IPv4, Site-to-Site, Networking, Linux

Description: Configure a WireGuard site-to-site VPN tunnel to interconnect two private IPv4 networks across the internet.

A site-to-site VPN connects two entire networks rather than individual clients. This is common when linking an office network to a data center or connecting two cloud VPCs. WireGuard handles this elegantly with a single peer configuration on each side.

## Topology

```text
Site A (Office):        Site B (Data Center):
192.168.1.0/24          192.168.2.0/24
       |                        |
  [Router A]  <--WireGuard-->  [Router B]
  WG IP: 10.10.0.1             WG IP: 10.10.0.2
  Public IP: 1.2.3.4           Public IP: 5.6.7.8
```

## Step 1: Configure Router A (Site A)

```ini
# /etc/wireguard/wg0.conf on Router A

[Interface]
PrivateKey = <ROUTER_A_PRIVATE_KEY>
# WireGuard tunnel IP for Router A

Address = 10.10.0.1/30
ListenPort = 51820

# Route traffic to Site B's LAN through the tunnel
PostUp = ip route add 192.168.2.0/24 dev wg0
PostDown = ip route del 192.168.2.0/24 dev wg0

[Peer]
PublicKey = <ROUTER_B_PUBLIC_KEY>
# Router B's public IP and WireGuard port
Endpoint = 5.6.7.8:51820
# Allow packets coming from Site B's WireGuard IP and its LAN subnet
AllowedIPs = 10.10.0.2/32, 192.168.2.0/24
PersistentKeepalive = 25
```

## Step 2: Configure Router B (Site B)

```ini
# /etc/wireguard/wg0.conf on Router B

[Interface]
PrivateKey = <ROUTER_B_PRIVATE_KEY>
Address = 10.10.0.2/30
ListenPort = 51820

PostUp = ip route add 192.168.1.0/24 dev wg0
PostDown = ip route del 192.168.1.0/24 dev wg0

[Peer]
PublicKey = <ROUTER_A_PUBLIC_KEY>
Endpoint = 1.2.3.4:51820
AllowedIPs = 10.10.0.1/32, 192.168.1.0/24
PersistentKeepalive = 25
```

## Step 3: Enable IP Forwarding on Both Routers

```bash
# On both Router A and Router B
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
```

## Step 4: Add Host Routes on LAN Devices

Devices on each LAN need to know to send traffic destined for the remote site through their local router. Either set a static route on each host or configure the default gateway's routing table.

```bash
# On a host in Site A (192.168.1.x), route to Site B via Router A
sudo ip route add 192.168.2.0/24 via 192.168.1.1

# Or configure the route in /etc/network/interfaces or netplan
```

## Step 5: Start WireGuard on Both Sides

```bash
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
```

## Verifying Connectivity

```bash
# From a host in Site A, ping a host in Site B
ping 192.168.2.10

# Check WireGuard handshake status on Router A
sudo wg show wg0
```

The `AllowedIPs` entries on each side must include both the tunnel IP (`/32`) of the remote peer and the remote LAN subnet. This is what causes WireGuard to route LAN-to-LAN traffic through the encrypted tunnel.
