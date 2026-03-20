# How to Set Up a WireGuard VPN Server with IPv4 on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: WireGuard, VPN, IPv4, Linux, Networking, Security

Description: A step-by-step guide to installing and configuring a WireGuard VPN server with IPv4 addressing on a Linux system.

WireGuard is a modern, high-performance VPN protocol built directly into the Linux kernel. It is far simpler to configure than OpenVPN or IPSec while offering excellent performance and a minimal attack surface.

## Prerequisites

- A Linux server (Ubuntu 20.04/22.04 recommended) with a public IPv4 address
- Root or sudo access
- A basic understanding of networking and firewalls

## Step 1: Install WireGuard

On Ubuntu/Debian:

```bash
# Update package list and install WireGuard
sudo apt update
sudo apt install wireguard -y
```

On RHEL/CentOS:

```bash
# Install EPEL and WireGuard
sudo dnf install epel-release -y
sudo dnf install wireguard-tools -y
```

## Step 2: Generate Server Keys

WireGuard uses Curve25519 asymmetric keys. Generate a private/public key pair for the server.

```bash
# Generate the private key and write it to a file with strict permissions
wg genkey | sudo tee /etc/wireguard/server_private.key | wg pubkey | sudo tee /etc/wireguard/server_public.key

# Secure the private key file
sudo chmod 600 /etc/wireguard/server_private.key
```

## Step 3: Create the Server Configuration

Replace `<SERVER_PRIVATE_KEY>` with the content of `/etc/wireguard/server_private.key`. The interface `wg0` will use `10.0.0.1/24` as its internal IPv4 address.

```ini
# /etc/wireguard/wg0.conf

[Interface]
# The private key for this server
PrivateKey = <SERVER_PRIVATE_KEY>

# The internal IPv4 address assigned to the VPN interface
Address = 10.0.0.1/24

# The UDP port WireGuard will listen on
ListenPort = 51820

# Save peer config changes made with wg set or wg addconf back to this file
SaveConfig = true

# Enable IP forwarding and NAT when the interface comes up
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
```

## Step 4: Enable IPv4 Forwarding

For the server to route traffic on behalf of clients, IPv4 forwarding must be enabled.

```bash
# Enable IP forwarding temporarily
sudo sysctl -w net.ipv4.ip_forward=1

# Make it permanent across reboots
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

## Step 5: Start WireGuard

```bash
# Start the WireGuard interface and enable it to start on boot
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0

# Verify the interface is up
sudo wg show
```

## Step 6: Open the Firewall Port

Allow inbound WireGuard traffic on UDP port 51820.

```bash
# Using ufw
sudo ufw allow 51820/udp

# Or using iptables directly
sudo iptables -A INPUT -p udp --dport 51820 -j ACCEPT
```

## Verifying the Setup

After starting WireGuard, you should see the `wg0` interface with your assigned address:

```bash
ip addr show wg0
# Expected output includes: inet 10.0.0.1/24
```

Your WireGuard server is now ready. The next step is to add client peers, each with their own key pair and an IP address within the `10.0.0.0/24` range.
