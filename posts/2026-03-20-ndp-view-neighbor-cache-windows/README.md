# How to View the IPv6 Neighbor Cache on Windows

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: NDP, Neighbor Cache, Window, Netsh, PowerShell

Description: View and manage the IPv6 neighbor cache on Windows using netsh and PowerShell commands, understand neighbor states, and clear stale entries.

## Introduction

Windows maintains an IPv6 neighbor cache similar to Linux's neighbor table. It maps IPv6 addresses to MAC addresses with NUD states. Windows uses `netsh` and PowerShell cmdlets to view and manage this cache. Understanding the neighbor cache is valuable for diagnosing IPv6 connectivity issues on Windows systems.

## Viewing the Neighbor Cache

```powershell
# PowerShell: Show all IPv6 neighbor cache entries

Get-NetNeighbor -AddressFamily IPv6

# Show for a specific interface
Get-NetNeighbor -AddressFamily IPv6 -InterfaceAlias "Ethernet"

# Show for a specific IP address
Get-NetNeighbor -IPAddress "2001:db8::1" -AddressFamily IPv6

# Show with state information
Get-NetNeighbor -AddressFamily IPv6 | Select-Object InterfaceAlias, IPAddress, LinkLayerAddress, State

# Filter by state
Get-NetNeighbor -AddressFamily IPv6 | Where-Object { $_.State -eq "Reachable" }
Get-NetNeighbor -AddressFamily IPv6 | Where-Object { $_.State -eq "Stale" }

# Show count by state
Get-NetNeighbor -AddressFamily IPv6 | Group-Object State | Sort-Object Count -Descending

# Using netsh (older but widely supported)
netsh interface ipv6 show neighbors
netsh interface ipv6 show neighbors interface="Ethernet"
```

## Neighbor States on Windows

```powershell
# Windows NUD states correspond to RFC 4861:
# Reachable  = REACHABLE (recently confirmed)
# Stale      = STALE (aged out, but cached)
# Delay      = DELAY (waiting for upper-layer confirmation)
# Probe      = PROBE (sending unicast NS probes)
# Unreachable = FAILED (all probes failed)
# Incomplete  = INCOMPLETE (resolution in progress)
# Permanent  = PERMANENT (static/manually added entry)

# View detailed state for all entries
Get-NetNeighbor -AddressFamily IPv6 | Format-Table -AutoSize

# Count entries by state
Get-NetNeighbor -AddressFamily IPv6 |
    Group-Object State |
    Select-Object Name, Count |
    Sort-Object Count -Descending
```

## Managing the Neighbor Cache

```powershell
# Delete a specific neighbor entry
Remove-NetNeighbor -IPAddress "2001:db8::1" -InterfaceAlias "Ethernet" -Confirm:$false

# Flush all IPv6 neighbor cache entries (all interfaces)
Get-NetNeighbor -AddressFamily IPv6 |
    Where-Object { $_.State -ne "Permanent" } |
    Remove-NetNeighbor -Confirm:$false

# Add a static (permanent) neighbor entry
New-NetNeighbor -InterfaceAlias "Ethernet" `
    -IPAddress "2001:db8::1" `
    -LinkLayerAddress "00-11-22-33-44-55" `
    -PolicyStore ActiveStore

# Using netsh to delete a neighbor
netsh interface ipv6 delete neighbors interface="Ethernet"

# View and delete specific entry with netsh
netsh interface ipv6 show neighbors interface="Ethernet"
```

## Scripted Neighbor Cache Analysis

```powershell
# PowerShell script to analyze neighbor cache health
function Analyze-IPv6NeighborCache {
    $neighbors = Get-NetNeighbor -AddressFamily IPv6

    $summary = $neighbors | Group-Object State |
        Select-Object @{N="State";E={$_.Name}},
                      @{N="Count";E={$_.Count}} |
        Sort-Object Count -Descending

    Write-Host "=== IPv6 Neighbor Cache Summary ==="
    $summary | Format-Table -AutoSize

    $failed = $neighbors | Where-Object { $_.State -eq "Unreachable" }
    if ($failed) {
        Write-Host "`n=== UNREACHABLE Neighbors (investigate these) ==="
        $failed | Select-Object InterfaceAlias, IPAddress | Format-Table -AutoSize
    }

    $incomplete = $neighbors | Where-Object { $_.State -eq "Incomplete" }
    if ($incomplete) {
        Write-Host "`n=== INCOMPLETE Neighbors (resolution in progress) ==="
        $incomplete | Select-Object InterfaceAlias, IPAddress | Format-Table -AutoSize
    }
}

Analyze-IPv6NeighborCache
```

## Conclusion

Windows provides two interfaces for IPv6 neighbor cache management: the PowerShell `Get-NetNeighbor` / `Remove-NetNeighbor` cmdlets and the legacy `netsh interface ipv6 show neighbors` command. The NUD states on Windows match the RFC 4861 state machine (Reachable, Stale, Delay, Probe, Unreachable). PowerShell cmdlets are preferred for scripting as they return structured objects. The neighbor cache is essential for diagnosing IPv6 connectivity: Unreachable entries indicate failed neighbor resolution, while Stale entries indicate aging but still-usable cache entries.
