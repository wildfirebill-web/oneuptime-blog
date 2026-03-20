# How to Troubleshoot Netplan Configuration Errors

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Netplan, Ubuntu, Troubleshooting, Networking, YAML

Description: Diagnose and fix Netplan configuration errors including YAML syntax issues, missing backend services, and network interface problems.

## Introduction

Netplan errors fall into categories: YAML syntax errors (caught by `netplan generate`), Netplan-specific errors (wrong keys or values), and runtime errors (interface not found, backend failing). Systematic diagnosis starts with validation and log review.

## Validate YAML Syntax

```bash
# Check for errors without applying
netplan generate

# Sample error output:
# Error in network definition /etc/netplan/01-netcfg.yaml line 5 column 3:
# unknown key 'addreses'
```

## Check Netplan Files for Common Issues

```bash
# List all Netplan files
ls -la /etc/netplan/

# Files must be owned by root
stat /etc/netplan/01-netcfg.yaml | grep "^Access"

# Fix ownership if wrong
chown root:root /etc/netplan/01-netcfg.yaml
chmod 600 /etc/netplan/01-netcfg.yaml
```

## View systemd-networkd Errors

```bash
# If using systemd-networkd backend
journalctl -u systemd-networkd -n 50

# Follow live
journalctl -u systemd-networkd -f

# Check interface status
networkctl status eth0
```

## View NetworkManager Errors

```bash
# If using NetworkManager backend
journalctl -u NetworkManager -n 50

# Check device status
nmcli device status
```

## Common Errors and Fixes

```bash
# Error: "interface eth0 not found"
# Fix: Check interface name
ip link show
# Interface names may be ens3, enp2s0, etc.
# Update the name in the Netplan file

# Error: "unknown key 'gateway4'"
# Fix: gateway4 is deprecated in newer Netplan
# Use routes: - to: default via: x.x.x.x

# Error: YAML indentation error
# Fix: Use 2-space indentation consistently
# Tab characters are NOT valid in YAML
```

## Debug with --debug Flag

```bash
netplan --debug apply 2>&1 | head -50
netplan --debug generate 2>&1
```

## Check Which Renderer is in Use

```yaml
# In your Netplan file, check renderer
network:
  version: 2
  renderer: networkd   # or NetworkManager
```

```bash
# Check if the correct backend is running
systemctl is-active systemd-networkd
systemctl is-active NetworkManager
```

## Recover from a Bad Configuration

```bash
# If network is broken after netplan apply
# Boot to recovery mode or use out-of-band access

# Remove or fix the bad config file
vim /etc/netplan/01-netcfg.yaml

# Or restore from backup
cp /etc/netplan/01-netcfg.yaml.bak /etc/netplan/01-netcfg.yaml

# Reapply
netplan apply
```

## Backup Before Changes

```bash
# Always backup before modifying Netplan
cp /etc/netplan/01-netcfg.yaml /etc/netplan/01-netcfg.yaml.bak
```

## Conclusion

Netplan troubleshooting starts with `netplan generate` to catch syntax errors, then `journalctl -u systemd-networkd` or `journalctl -u NetworkManager` for runtime errors. Common issues include wrong interface names, deprecated keys (`gateway4`), YAML indentation errors, and incorrect file permissions. Always back up before changing and use `netplan try` on remote servers.
