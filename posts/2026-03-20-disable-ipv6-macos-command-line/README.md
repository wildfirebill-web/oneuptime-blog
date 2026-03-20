# How to Disable IPv6 on macOS via Command Line

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, macOS, networksetup, Terminal, Disable IPv6

Description: Learn how to disable IPv6 on macOS using the networksetup command from Terminal, including disabling on all interfaces and making changes persistent.

## Disable IPv6 with networksetup

```bash
# Disable IPv6 on Wi-Fi
networksetup -setv6off Wi-Fi

# Disable IPv6 on Ethernet
networksetup -setv6off Ethernet

# Disable on Thunderbolt Bridge
networksetup -setv6off "Thunderbolt Bridge"

# List all network services to find names
networksetup -listallnetworkservices
```

## Disable IPv6 on All Interfaces

```bash
#!/bin/bash
# Disable IPv6 on all network services

echo "Disabling IPv6 on all network services..."

# Get list of all network services
networksetup -listallnetworkservices | tail -n +2 | while IFS= read -r service; do
    # Skip lines starting with * (disabled interfaces)
    [[ "$service" == \** ]] && continue

    echo "Disabling IPv6 on: $service"
    networksetup -setv6off "$service" 2>/dev/null
done

echo "Done. Verifying..."
networksetup -listallnetworkservices | tail -n +2 | while IFS= read -r service; do
    [[ "$service" == \** ]] && continue
    status=$(networksetup -getv6additional "$service" 2>/dev/null)
    echo "$service: $status"
done
```

## Set Link-Local Only (Partial Disable)

```bash
# Keep link-local IPv6 but disable global IPv6 on Wi-Fi
networksetup -setv6linklocal Wi-Fi

# This is useful when:
# - You need link-local for local network discovery
# - But don't want global IPv6 routing
```

## Verify IPv6 is Disabled

```bash
# Check IPv6 configuration for Wi-Fi
networksetup -getv6additional Wi-Fi
# Output: IPv6: Off

# Check via ifconfig
ifconfig en0 | grep inet6
# Should be empty (no inet6 lines) when disabled

# Confirm no global IPv6 routing
ping6 2001:4860:4860::8888
# Should fail when IPv6 is disabled
```

## Re-enable IPv6

```bash
# Re-enable automatic IPv6 on Wi-Fi
networksetup -setv6automatic Wi-Fi

# Re-enable on Ethernet
networksetup -setv6automatic Ethernet

# Verify addresses are assigned (may take a moment)
ifconfig en0 | grep inet6
```

## Using scutil for Advanced IPv6 Control

```bash
# Check IPv6 DNS configuration via scutil
scutil --dns | grep -A 5 "resolver #"

# Check network configuration
scutil --nwi

# These are read-only views; use networksetup for changes
```

## Persistent Disable via launchd (for headless/server macOS)

```bash
# Create a LaunchDaemon to disable IPv6 at boot
sudo tee /Library/LaunchDaemons/com.local.disable-ipv6.plist << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.local.disable-ipv6</string>
    <key>RunAtLoad</key>
    <true/>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/sbin/networksetup</string>
        <string>-setv6off</string>
        <string>Wi-Fi</string>
    </array>
</dict>
</plist>
EOF

sudo launchctl load /Library/LaunchDaemons/com.local.disable-ipv6.plist
```

## Summary

Disable IPv6 on macOS via command line with `networksetup -setv6off "Wi-Fi"` and `networksetup -setv6off "Ethernet"`. Use `networksetup -listallnetworkservices` to find exact interface names. For partial disable keeping link-local, use `networksetup -setv6linklocal`. Re-enable with `networksetup -setv6automatic`. For persistent disable across reboots on server macOS, use a LaunchDaemon.
