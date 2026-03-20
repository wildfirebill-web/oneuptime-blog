# How to Monitor Network Changes with nmcli monitor

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, nmcli, NetworkManager, Monitoring, Networking

Description: Monitor real-time network interface and connection changes using nmcli monitor, watching for events like connect, disconnect, IP address changes, and device state transitions.

## Introduction

`nmcli monitor` watches for NetworkManager events in real time, printing changes as they happen. This is useful for debugging connectivity issues, monitoring interface state transitions, and observing what happens when cables are plugged/unplugged or connections activate.

## Start Monitoring Network Events

```bash
# Watch all NetworkManager events
nmcli monitor

# Ctrl+C to stop
# Example output:
# eth0: connected to "Wired connection 1"
# "Wired connection 1" (ethernet, 192.168.1.100/24): connection activated
```

## Monitor a Specific Device

```bash
# Monitor events for eth0 only
nmcli device monitor eth0
```

## Monitor Device State Changes

```bash
# Watch for device state transitions
nmcli device monitor
# Shows: connected, disconnected, unavailable, unmanaged events
```

## Combine with journalctl for Detailed Logs

```bash
# Follow NetworkManager logs for more detail
journalctl -u NetworkManager -f

# Filter for connection events only
journalctl -u NetworkManager -f | grep -i "connected\|disconnected\|activated"
```

## Watch for IP Address Changes

```bash
# ip monitor shows kernel-level changes
ip monitor address

# Shows new/deleted IP addresses as they happen
```

## Watch for Route Changes

```bash
# Monitor routing table changes
ip monitor route

# Watch for both address and route changes
ip monitor all
```

## Scripted Monitoring

```bash
# Use nmcli in a loop to check connectivity
watch -n 5 'nmcli device status'

# Log connection events to a file
nmcli monitor >> /var/log/network-events.log 2>&1 &
```

## Check Current Device States

```bash
# Summary of all devices and their current state
nmcli device status

# Example output:
# DEVICE  TYPE      STATE      CONNECTION
# eth0    ethernet  connected  Wired connection 1
# eth1    ethernet  unavailable --
# lo      loopback  unmanaged  --
```

## Device State Meanings

| State | Description |
|---|---|
| `connected` | Interface is up and configured |
| `disconnected` | Interface is down |
| `unavailable` | No carrier (cable unplugged) |
| `unmanaged` | Not managed by NetworkManager |
| `connecting` | In progress |

## Conclusion

`nmcli monitor` and `nmcli device monitor` provide real-time visibility into NetworkManager events. For kernel-level changes (immediate, without NM processing), use `ip monitor`. Combine both with `journalctl -u NetworkManager -f` for comprehensive network change monitoring.
