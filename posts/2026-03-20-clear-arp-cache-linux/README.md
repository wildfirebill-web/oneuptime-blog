# How to Clear the ARP Cache

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Networking, ARP, Linux, Windows, macOS

Description: Learn how to clear the ARP cache on Linux, Windows, and macOS to resolve stale or incorrect ARP entries.

## Why Clear the ARP Cache?

You may need to clear the ARP cache when:
- A device's MAC address changed (hardware replacement)
- You're troubleshooting connectivity issues caused by stale entries
- You want to force fresh ARP resolution
- An IP address was reassigned to a new device

## Clearing ARP Cache on Linux

### Delete All Entries

```bash
# Flush all entries on eth0 (replace with your interface)
ip neigh flush dev eth0

# Flush all entries on all interfaces
ip neigh flush all

# Flush only failed entries
ip neigh flush nud failed

# Flush stale entries
ip neigh flush nud stale
```

### Delete a Specific Entry

```bash
# Remove a specific ARP entry
ip neigh del 192.168.1.20 dev eth0
```

### Verify the Flush

```bash
ip neigh show
# Should be empty or show only permanent entries
```

### Using the Legacy `arp` Command

```bash
# Delete a specific entry
arp -d 192.168.1.20

# Delete on a specific interface
arp -d 192.168.1.20 -i eth0
```

## Clearing ARP Cache on Windows

### Command Prompt

```cmd
# Delete all ARP entries
netsh interface ip delete arpcache

# Alternative using arp command
arp -d *
```

### PowerShell

```powershell
# Remove all dynamic ARP entries
Remove-NetNeighbor -Confirm:$false

# Remove entries for a specific interface
Remove-NetNeighbor -InterfaceAlias 'Ethernet' -Confirm:$false

# Remove a specific IP entry
Remove-NetNeighbor -IPAddress 192.168.1.20 -Confirm:$false
```

## Clearing ARP Cache on macOS

```bash
# Clear entire ARP cache (requires sudo)
sudo arp -a -d

# Delete a specific entry
sudo arp -d 192.168.1.20

# Alternative using ip (if installed)
sudo ip neigh flush all
```

## Script: Clear ARP Cache and Re-Ping

```bash
#!/bin/bash
# Clear ARP cache and force re-resolution for a host
TARGET="192.168.1.1"
IFACE="eth0"

echo "Removing ARP entry for $TARGET..."
ip neigh del "$TARGET" dev "$IFACE" 2>/dev/null

echo "Pinging $TARGET to trigger fresh ARP..."
ping -c 1 "$TARGET" > /dev/null 2>&1

echo "New ARP entry:"
ip neigh show "$TARGET"
```

## When ARP Cache Clears Automatically

ARP entries do not persist forever. On Linux:
- REACHABLE entries expire after ~30 seconds by default
- STALE entries are kept for ~60 seconds before being removed

Check the current timeouts:

```bash
sysctl net.ipv4.neigh.default.base_reachable_time_ms
sysctl net.ipv4.neigh.default.gc_stale_time
```

## Key Takeaways

- `ip neigh flush dev eth0` is the modern way to clear ARP on Linux.
- Windows uses `netsh interface ip delete arpcache` or PowerShell's `Remove-NetNeighbor`.
- macOS uses `sudo arp -a -d` to flush all entries.
- Clearing ARP cache forces fresh resolution and can fix stale-mapping issues.

**Related Reading:**

- [How to View the ARP Table on Linux](https://oneuptime.com/blog/post/2026-03-20-view-arp-table-linux/view)
- [How to Add Static ARP Entries](https://oneuptime.com/blog/post/2026-03-20-add-static-arp-entry-ip-neigh/view)
- [How to Set ARP Cache Timeouts on Linux](https://oneuptime.com/blog/post/2026-03-20-set-arp-cache-timeouts-linux/view)
