# How to Troubleshoot NetworkManager Connection Issues on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Networking, NetworkManager, nmcli, Troubleshooting, Connectivity

Description: Learn how to diagnose and fix NetworkManager connection problems on Linux using nmcli, journalctl, and other diagnostic tools to restore network connectivity.

## Introduction

NetworkManager manages network connections on most Linux desktop and server distributions. When connections fail or behave unexpectedly, understanding how to use `nmcli` and system logs helps you diagnose and fix issues quickly.

## Checking NetworkManager Status

```bash
systemctl status NetworkManager
```

If it's not running:

```bash
sudo systemctl start NetworkManager
sudo systemctl enable NetworkManager
```

## Listing Connections and Devices

```bash
# List all connections
nmcli connection show

# List device status
nmcli device status

# Show detailed device info
nmcli device show eth0
```

## Checking Connection State

```bash
nmcli -p connection show "MyConnection"
```

Look for:
- `GENERAL.STATE` - active/inactive
- `IP4.ADDRESS` - assigned IP
- `IP4.GATEWAY` - default gateway
- `IP4.DNS` - DNS servers

## Viewing Logs

```bash
journalctl -u NetworkManager --since "30 minutes ago" -f
```

Common log messages:
- `Activation failed` - connection could not activate
- `device (eth0): state change` - interface state transitions
- `dhcp4: timeout reached` - DHCP not responding

## Reconnecting a Connection

```bash
nmcli connection down "MyConnection"
nmcli connection up "MyConnection"
```

## Fixing DHCP Issues

If DHCP fails, manually release and renew:

```bash
nmcli device disconnect eth0
nmcli device connect eth0
```

Or force DHCP renewal:

```bash
sudo dhclient -r eth0
sudo dhclient eth0
```

## Fixing DNS Resolution

Check current DNS servers:

```bash
nmcli connection show "MyConnection" | grep DNS
resolvectl status eth0
```

Set custom DNS:

```bash
nmcli connection modify "MyConnection" ipv4.dns "8.8.8.8 1.1.1.1"
nmcli connection up "MyConnection"
```

## Profile Corruption

If a profile is corrupted, delete and recreate it:

```bash
nmcli connection delete "MyConnection"
nmcli connection add type ethernet ifname eth0 con-name "MyConnection" \
  ipv4.method auto
nmcli connection up "MyConnection"
```

## Checking Hardware

```bash
nmcli device show eth0 | grep GENERAL.HWADDR
ip link show eth0
ethtool eth0
```

## Resetting NetworkManager

As a last resort:

```bash
sudo systemctl restart NetworkManager
```

## Conclusion

NetworkManager troubleshooting follows a logical progression: check service status, review connection states, examine logs, and test connectivity at each layer. The `nmcli` tool provides everything needed to diagnose and fix most connection issues without GUI access.
