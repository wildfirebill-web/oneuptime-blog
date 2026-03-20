# How to Configure a Persistent Static Route on Windows

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Windows, Networking, route, Static Route, IPv4, Persistent, Network Configuration

Description: Add a persistent static IPv4 route on Windows that survives reboots using route add -p, New-NetRoute in PowerShell, and verify the route persists after a restart.

## Introduction

A persistent static route remains in the routing table after every reboot. Without the `-p` flag in `route add`, routes are lost when the machine restarts. Persistent routes are stored in the Windows registry and automatically loaded at startup.

## Adding a Persistent Route with route add

```cmd
:: The -p flag makes the route persistent
route add -p 10.0.0.0 mask 255.0.0.0 192.168.1.254

:: Add with explicit metric
route add -p 172.16.0.0 mask 255.240.0.0 192.168.1.254 metric 5

:: Verify immediately
route print | findstr "10.0.0"
```

## Confirming Persistence in the Persistent Routes Section

```cmd
:: Check the Persistent Routes section at the bottom of route print
route print -4

:: Look for:
:: Persistent Routes:
::   Network Address    Netmask           Gateway Address  Metric
::        10.0.0.0   255.0.0.0           192.168.1.254       1
```

## Using PowerShell New-NetRoute (Always Persistent)

Routes added with `New-NetRoute` in PowerShell are automatically persistent:

```powershell
# Add persistent static route
New-NetRoute `
    -InterfaceAlias "Ethernet" `
    -DestinationPrefix "10.0.0.0/8" `
    -NextHop "192.168.1.254" `
    -RouteMetric 5

# Verify
Get-NetRoute -DestinationPrefix "10.0.0.0/8"
```

## Verifying After Reboot

After creating persistent routes, test them by rebooting and checking:

```cmd
:: After restart, verify the route is still present
route print -4 | findstr "10.0.0"
```

## Registry Location of Persistent Routes

Persistent routes are stored here (for reference/audit):

```
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters\PersistentRoutes
```

Do not modify the registry directly — use `route add -p` or PowerShell.

## Bulk Persistent Route Script

```cmd
@echo off
:: Add multiple persistent routes at once
route add -p 10.0.0.0      mask 255.0.0.0       192.168.1.254 metric 5
route add -p 172.16.0.0    mask 255.240.0.0      192.168.1.254 metric 5
route add -p 192.168.100.0 mask 255.255.255.0    192.168.1.253 metric 5
echo Persistent routes added.
route print -4
```

## Removing a Persistent Route

```cmd
:: Removes from both active and persistent tables
route delete 10.0.0.0 mask 255.0.0.0

:: Via PowerShell
Remove-NetRoute -DestinationPrefix "10.0.0.0/8" -Confirm:$false
```

## Conclusion

The `-p` flag in `route add` is the only difference between a temporary and persistent route. Always verify with `route print` and look at the **Persistent Routes** section. For scripted deployments, PowerShell's `New-NetRoute` provides cleaner syntax and is always persistent.
