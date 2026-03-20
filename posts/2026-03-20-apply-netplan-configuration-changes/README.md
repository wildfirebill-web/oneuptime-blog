# How to Apply Netplan Configuration Changes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Netplan, Ubuntu, Networking, Configuration

Description: Apply Netplan network configuration changes using netplan apply, netplan try, and netplan generate, understanding the difference between each command.

## Introduction

After modifying `.yaml` files in `/etc/netplan/`, you must apply the changes to make them active. Netplan provides three main commands: `netplan generate` (validates and generates backend configs), `netplan apply` (applies immediately), and `netplan try` (applies with auto-revert).

## Apply Changes Immediately

```bash
# Apply all Netplan configurations immediately
netplan apply

# Verify changes took effect
ip addr show
ip route show
```

## Using netplan try (Recommended for Remote Servers)

```bash
# Apply with a 120-second timeout
# If you don't confirm, it auto-reverts
netplan try

# You'll see:
# "Do you want to keep these settings?
# Press ENTER before the timeout to accept the new configuration"
```

This is safer for remote SSH sessions — if the new config breaks connectivity, the old config is automatically restored after 120 seconds.

## Generate Backend Config Without Applying

```bash
# Validate YAML and generate backend config files (no apply)
netplan generate

# Generated files go to /run/systemd/network/ or /run/NetworkManager/
ls /run/systemd/network/
```

## Validate YAML Before Applying

```bash
# Check for syntax errors
netplan generate 2>&1

# A clean validation produces no output (exit code 0)
echo $?
```

## Apply with Verbose Output

```bash
# See what netplan is doing during apply
netplan --debug apply

# Or
netplan apply -d
```

## Apply Only Specific Configuration File

```bash
# Netplan processes all files in /etc/netplan/ alphabetically
# You cannot apply a single file — all files are applied together

# To isolate testing, temporarily remove other files or use netplan try
```

## Reload vs Apply

```bash
# netplan apply: full reconfiguration (interfaces may briefly go down)
netplan apply

# networkctl reload: for systemd-networkd backend (only reloads .network files)
networkctl reload
```

## Check Netplan Version

```bash
netplan --version
```

## Common Apply Issues

```bash
# Issue: YAML syntax error
netplan generate
# Output shows line/column of error

# Issue: Changes not taking effect
# Check which backend is in use
cat /etc/netplan/01-*.yaml | grep renderer

# Default renderer is systemd-networkd on Ubuntu Server
# renderer: NetworkManager uses NetworkManager backend
```

## Conclusion

Use `netplan try` for safe interactive testing (auto-reverts if you don't confirm), `netplan apply` for scripted/immediate application, and `netplan generate` for validation without applying. After applying, verify with `ip addr`, `ip route`, and `ping` to confirm the configuration is working.
