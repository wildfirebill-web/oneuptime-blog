# How to Configure IPv6 DNS Servers on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, DNS, Linux, resolv.conf, Network Configuration

Description: Learn how to configure IPv6 DNS server addresses on Linux using resolv.conf, systemd-resolved, NetworkManager, and Netplan.

## Method 1: Direct resolv.conf Configuration

The simplest way to set DNS servers including IPv6:

```bash
# Edit /etc/resolv.conf

cat > /etc/resolv.conf << 'EOF'
# IPv6 DNS servers (Google Public DNS)
nameserver 2001:4860:4860::8888
nameserver 2001:4860:4860::8844

# IPv4 DNS fallback
nameserver 8.8.8.8
nameserver 8.8.4.4

# Search domain
search example.com
EOF
```

Note: On systems managed by NetworkManager or systemd-resolved, `/etc/resolv.conf` is a symlink and will be overwritten. Use the manager-specific methods below.

## Method 2: systemd-resolved Configuration

```bash
# Check current DNS servers
resolvectl status

# Configure IPv6 DNS globally
cat > /etc/systemd/resolved.conf << 'EOF'
[Resolve]
DNS=2001:4860:4860::8888 2001:4860:4860::8844 8.8.8.8 8.8.4.4
FallbackDNS=2620:fe::fe 9.9.9.9
Domains=example.com
EOF

# Restart systemd-resolved
systemctl restart systemd-resolved

# Verify new DNS settings
resolvectl status
resolvectl dns

# Test IPv6 DNS resolution
resolvectl query -6 google.com
```

## Method 3: NetworkManager (RHEL/Ubuntu with NM)

```bash
# Set IPv6 DNS for a specific connection
nmcli connection modify eth0 \
    ipv6.dns "2001:4860:4860::8888 2001:4860:4860::8844"

# Apply changes
nmcli connection up eth0

# Verify
nmcli connection show eth0 | grep ipv6.dns
```

## Method 4: Netplan (Ubuntu)

```yaml
# /etc/netplan/00-installer-config.yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: true
      accept-ra: true
      nameservers:
        addresses:
          - 2001:4860:4860::8888    # Google IPv6 DNS primary
          - 2001:4860:4860::8844    # Google IPv6 DNS secondary
          - 8.8.8.8                 # IPv4 fallback
        search:
          - example.com
```

```bash
netplan apply
```

## Method 5: Interface-Specific DNS via systemd-networkd

```ini
# /etc/systemd/network/10-eth0.network

[Match]
Name=eth0

[Network]
Address=2001:db8::10/64
Gateway=2001:db8::1
DNS=2001:4860:4860::8888
DNS=2001:4860:4860::8844
DNS=8.8.8.8
Domains=example.com
```

## Well-Known IPv6 DNS Servers

```text
Google Public DNS:
  Primary:   2001:4860:4860::8888
  Secondary: 2001:4860:4860::8844

Cloudflare DNS:
  Primary:   2606:4700:4700::1111
  Secondary: 2606:4700:4700::1001

Quad9 DNS:
  Primary:   2620:fe::fe
  Secondary: 2620:fe::9

OpenDNS:
  Primary:   2620:119:35::35
  Secondary: 2620:119:53::53
```

## Verifying DNS Resolution over IPv6

```bash
# Test that DNS resolver is accessible via IPv6
dig A google.com @2001:4860:4860::8888

# Test AAAA resolution
dig AAAA google.com @2001:4860:4860::8888

# Check which DNS servers are being used
resolvectl status | grep 'DNS Server'

# Verify DNS transport is over IPv6 (check with tcpdump)
tcpdump -i eth0 -n 'port 53 and ip6' &
dig AAAA google.com
fg; # Ctrl+C
```

## Summary

Configure IPv6 DNS servers via `/etc/resolv.conf` (immediate but non-persistent on managed systems), `systemd-resolved` (edit `/etc/systemd/resolved.conf`), NetworkManager (`nmcli connection modify ipv6.dns`), or Netplan (`nameservers.addresses`). Use Google (`2001:4860:4860::8888`), Cloudflare (`2606:4700:4700::1111`), or Quad9 (`2620:fe::fe`) as trusted IPv6 DNS providers. Verify with `resolvectl status` and `dig @<ipv6-dns-server>`.
