# How to Migrate from NetworkManager to systemd-networkd

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, systemd-networkd, NetworkManager, Migration, Networking, Configuration

Description: Migrate network configuration from NetworkManager to systemd-networkd, converting existing connections to .network and .netdev files.

## Introduction

Migrating from NetworkManager to systemd-networkd involves: documenting current network configuration, disabling NetworkManager, creating equivalent `.network` files, and verifying connectivity. This is commonly done on servers where NetworkManager's desktop-oriented features are unnecessary.

## Step 1: Document Current Configuration

```bash
# List all NetworkManager connections
nmcli connection show

# Get details for each connection
nmcli connection show "Wired connection 1"

# Show current IP configuration
ip addr show
ip route show

# Show current DNS
cat /etc/resolv.conf
```

## Step 2: Create systemd-networkd .network Files

For a static IP connection replacing NM connection on eth0:

```ini
# /etc/systemd/network/10-eth0.network
[Match]
Name=eth0

[Network]
Address=192.168.1.100/24
Gateway=192.168.1.1
DNS=8.8.8.8
DNS=8.8.4.4
```

For DHCP:

```ini
# /etc/systemd/network/10-eth0.network
[Match]
Name=eth0

[Network]
DHCP=ipv4
```

## Step 3: Enable systemd-networkd and systemd-resolved

```bash
# Enable services
systemctl enable systemd-networkd
systemctl enable systemd-resolved

# Configure /etc/resolv.conf for systemd-resolved
ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

## Step 4: Disable NetworkManager

```bash
# Stop and disable NetworkManager
systemctl stop NetworkManager
systemctl disable NetworkManager

# Prevent it from starting (optional but recommended)
systemctl mask NetworkManager
```

## Step 5: Start systemd-networkd

```bash
# Start the new network manager
systemctl start systemd-networkd
systemctl start systemd-resolved
```

## Step 6: Verify

```bash
# Check all interfaces are managed and configured
networkctl list

# Check individual interface
networkctl status eth0

# Verify IP and routing
ip addr show
ip route show

# Test DNS
resolvectl query google.com
```

## Rollback Plan

```bash
# If something goes wrong, re-enable NetworkManager
systemctl unmask NetworkManager
systemctl enable --now NetworkManager
systemctl stop systemd-networkd
```

## Common Migration Issues

- Interface names may differ between tools — use `ip link show` to confirm
- NetworkManager stores WiFi passwords; systemd-networkd requires separate WPA supplicant config for wireless
- VPN connections may need reconfiguration

## Conclusion

Migrating from NetworkManager to systemd-networkd involves creating `.network` files mirroring current NM connection settings, enabling both `systemd-networkd` and `systemd-resolved`, then stopping and disabling NetworkManager. Test thoroughly before finalizing, and keep a rollback plan accessible.
