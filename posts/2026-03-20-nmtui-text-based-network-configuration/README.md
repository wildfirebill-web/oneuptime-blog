# How to Use nmtui for Text-Based Network Configuration - Configuration

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: nmtui, NetworkManager, Linux, Network Configuration, System Administration

Description: Learn how to use nmtui (NetworkManager Text User Interface) to configure network connections, IP addresses, and Wi-Fi on Linux systems without a graphical desktop.

## Introduction

nmtui is a curses-based text user interface for NetworkManager, available on most modern Linux distributions. It provides an interactive menu for configuring network connections without needing a GUI or memorizing complex `nmcli` syntax - ideal for server environments and remote SSH sessions.

## Prerequisites

- A Linux system with NetworkManager installed
- Terminal access (local or SSH)
- Root or sudo privileges

## Installing NetworkManager

```bash
# Ubuntu/Debian

apt-get install network-manager

# RHEL/CentOS/Fedora
dnf install NetworkManager

# Enable and start
systemctl enable --now NetworkManager
```

## Launching nmtui

```bash
nmtui
```

The main menu offers three options:
1. **Edit a connection** - configure existing connections
2. **Activate a connection** - bring connections up or down
3. **Set system hostname** - change the hostname

Navigate with arrow keys, select with Enter, and go back with Escape.

## Editing a Network Connection

### Static IPv4 Configuration

1. Select **Edit a connection**.
2. Select your interface (e.g., `eth0` or `ens3`) and press **Edit**.
3. Under **IPv4 CONFIGURATION**, change `Automatic` to `Manual`.
4. Select **Show** to expand the fields.
5. Fill in:
   - **Addresses**: `192.168.1.100/24`
   - **Gateway**: `192.168.1.1`
   - **DNS servers**: `8.8.8.8 8.8.4.4`
   - **Search domains**: `example.com`
6. Select **OK** to save.

### Setting IPv6 Configuration

1. Under **IPv6 CONFIGURATION**, change to `Manual`.
2. Select **Show**.
3. Fill in:
   - **Addresses**: `2001:db8::10/64`
   - **Gateway**: `2001:db8::1`
4. Select **OK**.

### Configuring DHCP

1. Under **IPv4 CONFIGURATION**, select `Automatic` (DHCP).
2. No further fields are required.
3. Optionally configure **DNS servers** to override DHCP-provided DNS.

## Connecting to Wi-Fi

1. Launch `nmtui`.
2. Select **Activate a connection**.
3. Find your Wi-Fi network under **Wi-Fi**.
4. Select it and press **Activate**.
5. Enter the Wi-Fi password when prompted.

Or configure a new Wi-Fi connection:

1. **Edit a connection** > **Add**.
2. Select **Wi-Fi** as the connection type.
3. Fill in the SSID and security credentials.

## Setting the System Hostname

1. Select **Set system hostname** from the main menu.
2. Enter the new hostname.
3. Select **OK**.

Apply immediately:

```bash
hostnamectl set-hostname new-hostname
```

## Activating and Deactivating Connections

1. From the main menu, select **Activate a connection**.
2. Scroll to find the connection.
3. Press Enter to activate (shows `*` prefix when active).
4. Press Enter again to deactivate.

## Verifying Changes

After configuring through nmtui:

```bash
# Check current IP configuration
ip addr show

# Verify routing table
ip route show

# Test DNS resolution
nslookup example.com

# Check NetworkManager status
nmcli connection show
```

## Applying Changes Without Reboot

Changes made in nmtui take effect when you activate the connection. If the connection was already active:

```bash
# Restart the specific connection
nmcli connection down eth0 && nmcli connection up eth0

# Or restart NetworkManager (brief network interruption)
systemctl restart NetworkManager
```

## Common nmtui Scenarios

### Configure a Second IP Address

1. Edit the connection.
2. Under IPv4, add a second entry to **Addresses**:
   - `192.168.1.100/24`
   - `192.168.1.101/24`

### Set DNS Search Domains

In the **Search domains** field, add your domain:
- `example.com`
- `internal.example.com`

### Configure a VLAN Interface

1. **Edit a connection** > **Add**.
2. Choose **VLAN** as the connection type.
3. Set the parent interface and VLAN ID.
4. Configure the IP address.

## nmtui vs nmcli

| Use Case | Tool |
|---|---|
| Interactive configuration | nmtui |
| Scripted/automated config | nmcli |
| Quick status check | nmcli |
| Remote session without expertise | nmtui |

## Best Practices

- Use nmtui for one-off configurations; use nmcli scripts for repeatable automation.
- Always verify changes with `ip addr show` after applying.
- Take note of current configuration before editing in case you need to revert.
- For servers, use static IPs rather than DHCP for predictable addressing.
- Back up `/etc/NetworkManager/system-connections/` before making changes.

## Conclusion

nmtui makes network configuration accessible on headless Linux systems through an intuitive text menu. Whether configuring static IPs, DNS, Wi-Fi, or VLANs, nmtui provides a guided interface that reduces the risk of syntax errors common with manual configuration file editing.
