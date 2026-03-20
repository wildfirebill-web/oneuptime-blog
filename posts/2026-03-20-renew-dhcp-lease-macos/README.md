# How to Renew a DHCP Lease on macOS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DHCP, macOS, Networking, Network Diagnostics, Sysadmin

Description: Renewing a DHCP lease on macOS can be done through System Settings, the ipconfig command, or by using networksetup to toggle DHCP off and on for the target interface.

## Method 1: System Settings (GUI)

1. Open **System Settings → Network**.
2. Select the active network interface (e.g., Wi-Fi or Ethernet).
3. Click **Details** (macOS Ventura+).
4. Go to the **TCP/IP** tab.
5. Click **Renew DHCP Lease**.
6. Click **OK**.

## Method 2: ipconfig Command

```bash
# Show current DHCP lease information

ipconfig getpacket en0

# Release and renew the DHCP lease
sudo ipconfig set en0 DHCP

# For Wi-Fi (en0) or Ethernet (en1) - check with ifconfig
ifconfig | grep '^en' | awk '{print $1}'

# Verbose renewal
sudo ipconfig -v setifaddr en0
```

## Method 3: networksetup (CLI)

```bash
# Toggle from DHCP to DHCP to force renewal
sudo networksetup -setdhcp "Wi-Fi"

# Get current IP info for the interface
networksetup -getinfo "Wi-Fi"
```

## Method 4: Disable and Re-Enable Interface

```bash
# Bring down and bring up the interface
sudo ifconfig en0 down
sleep 2
sudo ifconfig en0 up

# Or via networksetup
sudo networksetup -setnetworkserviceenabled "Wi-Fi" off
sleep 2
sudo networksetup -setnetworkserviceenabled "Wi-Fi" on
```

## Viewing Current Lease Details

```bash
# Detailed DHCP packet info (shows server IP, lease time, all options)
ipconfig getpacket en0

# Current IP configuration
networksetup -getinfo "Wi-Fi"

# Or
ifconfig en0
```

## Flushing DNS Cache After Renewal

```bash
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
```

## Troubleshooting

```bash
# Check DHCP logs in Console app or:
log show --predicate 'process == "configd"' --last 1h | grep -i dhcp

# Verify which DHCP server responded
ipconfig getpacket en0 | grep server_identifier
```

## Key Takeaways

- The GUI renewal button in System Settings is the simplest method for most users.
- `sudo ipconfig set en0 DHCP` triggers a new DORA exchange from the command line.
- Use `ipconfig getpacket en0` to see all DHCP options including the server IP and lease time.
- Flush the DNS cache with `dscacheutil -flushcache` after renewal to clear stale records.
