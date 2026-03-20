# How to Configure Tailscale with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Tailscale, IPv6, VPN, WireGuard, Mesh Network, Zero-Config

Description: A guide to Tailscale's IPv6 support, including how Tailscale assigns IPv6 addresses to devices, configures dual-stack connectivity, and handles IPv6 exit nodes.

Tailscale is a mesh VPN built on WireGuard that automatically handles IPv6 in several ways: assigning IPv6 addresses within the Tailscale network (from the `fd7a:115c:a1e0::/48` range), supporting IPv6 as a transport protocol for WireGuard connections, and providing IPv6 exit nodes.

## Tailscale IPv6 Address Assignment

Every Tailscale device automatically receives an IPv6 address:

```
IPv4 Tailscale address: 100.x.x.x (Carrier-Grade NAT range)
IPv6 Tailscale address: fd7a:115c:a1e0::/48 (unique-local range)
```

```bash
# After installing Tailscale, check assigned addresses
tailscale ip

# Example output:
# 100.101.102.103          (IPv4)
# fd7a:115c:a1e0:ab12::1   (IPv6)

# Or check via ip command
ip addr show tailscale0
```

## Installing Tailscale

```bash
# Linux (official script)
curl -fsSL https://tailscale.com/install.sh | sh

# Debian/Ubuntu manual
curl -fsSL https://pkgs.tailscale.com/stable/debian/bookworm.noarmor.gpg | \
  sudo tee /usr/share/keyrings/tailscale-archive-keyring.gpg >/dev/null

echo "deb [signed-by=/usr/share/keyrings/tailscale-archive-keyring.gpg] \
  https://pkgs.tailscale.com/stable/debian bookworm main" | \
  sudo tee /etc/apt/sources.list.d/tailscale.list

sudo apt-get update && sudo apt-get install tailscale

# Start and authenticate
sudo tailscale up
```

## Enabling IPv6 Transport

Tailscale uses IPv6 transport automatically when available:

```bash
# Start Tailscale with verbose logging to see transport used
sudo tailscale up --verbose

# Check if peers are connected via IPv6
tailscale status

# Tailscale will show something like:
# 100.x.x.x  hostname  active; relay "nyc", tx 1.5MB, rx 2.3MB
# or
# 100.x.x.x  hostname  active; direct 2001:db8::peer:41641, tx ...
# "direct 2001:db8::..." means IPv6 direct connection
```

## Pinging Over IPv6 with Tailscale

```bash
# Ping via Tailscale IPv6 address
ping6 fd7a:115c:a1e0::peer-address

# Or use the hostname (resolves to both IPv4 and IPv6 Tailscale addresses)
ping6 hostname.tailnet-name.ts.net

# Check MagicDNS resolves IPv6
dig AAAA hostname.tailnet-name.ts.net
```

## IPv6 Exit Node Configuration

An exit node routes all your traffic through another Tailscale device, including IPv6:

```bash
# On the exit node: advertise as exit node
sudo tailscale up --advertise-exit-node

# On client: use the exit node
sudo tailscale up --exit-node=100.x.x.exit-node
# or by name
sudo tailscale up --exit-node=exit-node-hostname

# Enable IPv6 routing through exit node
sudo tailscale up --exit-node=exit-node-hostname --exit-node-allow-lan-access=false
```

## Subnet Routing with IPv6

Tailscale can route to subnets beyond the tailnet, including IPv6 subnets:

```bash
# Advertise an IPv6 subnet as accessible via this Tailscale node
sudo tailscale up --advertise-routes=2001:db8:internal::/48

# On the Tailscale admin panel: approve the subnet route
# https://login.tailscale.com/admin/machines

# On clients: enable subnet routing
sudo tailscale up --accept-routes
```

## Verifying Dual-Stack Tailscale

```bash
# Check all Tailscale IPs
tailscale ip -4    # IPv4 only
tailscale ip -6    # IPv6 only

# Test connectivity to another device using IPv6
tailscale ping --peerapi fd7a:115c:a1e0::peer

# View network status including IPv6 connections
tailscale status --peers
```

## DNS and IPv6 with Tailscale MagicDNS

```bash
# MagicDNS automatically creates AAAA records for Tailscale devices
dig AAAA my-server.tailnet-name.ts.net

# Verify DNS resolution returns IPv6 Tailscale address
nslookup -type=AAAA my-server.tailnet-name.ts.net 100.100.100.100
# 100.100.100.100 is Tailscale's MagicDNS resolver
```

Tailscale's automatic IPv6 address assignment and transparent IPv6 transport support means most IPv6 features work without any additional configuration — Tailscale handles the addressing and routing automatically.
