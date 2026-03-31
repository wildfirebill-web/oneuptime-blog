# How to Troubleshoot NetworkManager Connection Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, NetworkManager, nmcli, Troubleshooting, Networking, Diagnostic

Description: Troubleshoot NetworkManager connection problems using nmcli, journalctl logs, and network diagnostic tools to diagnose and resolve common issues.

## Introduction

NetworkManager issues typically fall into categories: interface not connecting, wrong IP assignment, DNS failures, or connection not persisting. Systematic use of `nmcli`, logs, and network tools narrows down the root cause.

## Check NetworkManager Service Status

```bash
# Is NetworkManager running?

systemctl status NetworkManager

# Restart if needed
systemctl restart NetworkManager
```

## List All Devices and Their States

```bash
# Quick overview of all devices
nmcli device status

# Look for: unavailable, disconnected, unmanaged
```

## Check Specific Connection Status

```bash
# Show active connections
nmcli connection show --active

# Show all configured connections
nmcli connection show

# Detailed info for a specific connection
nmcli connection show "Wired connection 1"
```

## View NetworkManager Logs

```bash
# Recent logs
journalctl -u NetworkManager -n 100

# Follow live logs
journalctl -u NetworkManager -f

# Filter for errors
journalctl -u NetworkManager | grep -i error
```

## Enable Verbose Logging

```bash
# Set NetworkManager to debug level temporarily
nmcli general logging level DEBUG domains ALL

# Check logs
journalctl -u NetworkManager -f

# Reset to default logging
nmcli general logging level INFO domains ALL
```

## Diagnose Connection Failures

```bash
# Manually activate a connection and watch for errors
nmcli connection up "myconn" --wait 30

# Force reconnect a device
nmcli device reapply eth0

# Disconnect and reconnect
nmcli device disconnect eth0
nmcli device connect eth0
```

## Check for Conflicting Configurations

```bash
# Ensure no other network manager is running
systemctl status networking
systemctl status ifupdown
systemctl status systemd-networkd

# Multiple managers can conflict
```

## Common Issues and Fixes

```bash
# Issue: Interface shows "unmanaged"
# Fix: Check if interface is listed in /etc/NetworkManager/conf.d/
grep -r "unmanaged-devices" /etc/NetworkManager/

# Issue: DHCP not getting an address
# Fix: Check DHCP client logs
journalctl -u NetworkManager | grep -i dhcp

# Issue: DNS not resolving after connect
# Fix: Check resolv.conf and DNS settings
cat /etc/resolv.conf
nmcli device show eth0 | grep DNS
```

## Check /etc/resolv.conf State

```bash
# If DNS is broken, check what resolv.conf points to
ls -la /etc/resolv.conf

# Should be a symlink to:
# /run/systemd/resolve/stub-resolv.conf (with systemd-resolved)
# /run/NetworkManager/resolv.conf (direct NM)
```

## Conclusion

Start NetworkManager troubleshooting with `nmcli device status` and `journalctl -u NetworkManager`. Enable debug logging for detailed event traces. Common issues involve conflicting network managers, missing connection profiles, or DHCP failures visible in the logs. Always verify the state with both `nmcli` and `ip addr`/`ip route` to compare configured vs actual state.
