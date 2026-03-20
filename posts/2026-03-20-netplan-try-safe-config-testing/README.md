# How to Use netplan try for Safe Configuration Testing

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Netplan, Ubuntu, Networking, Configuration, Safety

Description: Use netplan try to safely test Netplan configuration changes with automatic rollback, preventing loss of remote access if a new configuration breaks connectivity.

## Introduction

`netplan try` applies network configuration changes temporarily. If you don't confirm within 120 seconds (default), the previous configuration is automatically restored. This is essential for remote servers where a bad network config would lock you out.

## Basic Usage

```bash
# Apply configuration with automatic rollback
netplan try

# Netplan shows:
# "Do you want to keep these settings?
#  Press ENTER before the timeout to accept the new configuration
# Changes will revert in 120 seconds"

# Press ENTER to confirm and make changes permanent
# Wait or lose connectivity → automatic rollback
```

## Custom Timeout

```bash
# Set a 60-second timeout instead of default 120
netplan try --timeout 60

# For extended testing (5 minutes)
netplan try --timeout 300
```

## Workflow for Remote Server Changes

```bash
# Step 1: Make changes to Netplan config
vim /etc/netplan/01-netcfg.yaml

# Step 2: Validate syntax first
netplan generate

# Step 3: Apply with automatic rollback
netplan try

# Step 4: Test connectivity (new SSH session recommended)
# From another terminal: ping <server-ip>

# Step 5: If working, confirm in the original session
# Press ENTER

# If broken, do nothing — it auto-reverts after timeout
```

## netplan try vs netplan apply

| Feature | netplan try | netplan apply |
|---|---|---|
| Rollback | Automatic (120s) | None |
| Confirm required | Yes | No |
| Safe for remote | Yes | Risky |
| Scripting | No (interactive) | Yes |

## Testing a New Interface Configuration

```yaml
# Edit: /etc/netplan/01-netcfg.yaml
# Changed from DHCP to static
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: false
      addresses:
        - 10.0.0.50/24
      routes:
        - to: default
          via: 10.0.0.1
```

```bash
# Test safely
netplan try

# Open new SSH to 10.0.0.50 to verify
# Confirm if working, do nothing if not
```

## Automate Confirmation (For Testing)

```bash
# Automatically confirm after 10 seconds (useful in scripts)
echo "" | timeout 10 netplan try || true
# Note: This bypasses the safety; use only in controlled environments
```

## Conclusion

`netplan try` is the safest way to apply Netplan changes on remote servers. It applies changes immediately but auto-reverts to the previous configuration if not confirmed within the timeout. Always use `netplan try` before `netplan apply` on production servers — one lost SSH session is worth the extra step.
