# How to Configure WireGuard VPN with Netplan on Ubuntu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, WireGuard, VPN, Netplan, Ubuntu, Networking

Description: Learn how to configure WireGuard VPN tunnels using Netplan on Ubuntu, providing a clean YAML-based configuration that integrates with the systemd-networkd backend.

## Introduction

WireGuard is a modern, fast, and secure VPN protocol. Ubuntu's Netplan network configuration system supports WireGuard natively when using the `systemd-networkd` backend, allowing you to define VPN tunnels using familiar YAML configuration files.

## Prerequisites

- Ubuntu 20.04 or later with systemd-networkd backend
- WireGuard installed
- A WireGuard key pair (public and private)

## Installing WireGuard

```bash
sudo apt update
sudo apt install wireguard
```

## Generating Key Pairs

On both the server and client:

```bash
wg genkey | sudo tee /etc/wireguard/private.key
sudo chmod 600 /etc/wireguard/private.key
sudo cat /etc/wireguard/private.key | wg pubkey | sudo tee /etc/wireguard/public.key
```

## Netplan WireGuard Configuration

Create or edit your Netplan configuration:

```bash
sudo nano /etc/netplan/50-wireguard.yaml
```

**Server configuration:**

```yaml
network:
  version: 2
  tunnels:
    wg0:
      mode: wireguard
      port: 51820
      key: /etc/wireguard/private.key
      addresses:
        - 10.100.0.1/24
      peers:
        - keys:
            public: <client-public-key>
          allowed-ips:
            - 10.100.0.2/32
          keepalive: 25
```

**Client configuration:**

```yaml
network:
  version: 2
  tunnels:
    wg0:
      mode: wireguard
      key: /etc/wireguard/private.key
      addresses:
        - 10.100.0.2/24
      routes:
        - to: 0.0.0.0/0
          via: 10.100.0.1
      peers:
        - keys:
            public: <server-public-key>
          endpoint: server.example.com:51820
          allowed-ips:
            - 0.0.0.0/0
          keepalive: 25
```

## Applying the Configuration

```bash
sudo netplan apply
```

## Verifying WireGuard Status

```bash
sudo wg show
ip link show wg0
ip -4 addr show wg0
```

## Testing the VPN

From the client:

```bash
ping 10.100.0.1
```

From the server:

```bash
ping 10.100.0.2
```

## Enabling IP Forwarding on the Server

For routing traffic through the VPN:

```bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

## Conclusion

Netplan's WireGuard support provides a clean, YAML-based way to manage VPN tunnels that integrates naturally with Ubuntu's network configuration ecosystem. The configuration persists across reboots and works well with both the systemd-networkd backend.
