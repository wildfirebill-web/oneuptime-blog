# How to Display the Routing Table with route print on Windows

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Windows, Networking, route, Routing Table, IPv4, Diagnostics

Description: Display and interpret the Windows IPv4 routing table using route print, understand the output columns, and filter for specific routes.

## Introduction

`route print` displays the complete Windows routing table. Understanding its output lets you verify that default routes, static routes, and connected-network routes are correctly configured.

## Displaying the Full Routing Table

```cmd
route print
```

The output has three sections:
1. **Interface List**: network adapters and their interface indices
2. **IPv4 Route Table**: all IPv4 routes
3. **IPv6 Route Table**: all IPv6 routes

## Displaying Only IPv4 Routes

```cmd
route print -4
```

## Understanding the IPv4 Route Table Columns

```
IPv4 Route Table
===========================================================================
Active Routes:
Network Destination  Netmask        Gateway       Interface   Metric
        0.0.0.0      0.0.0.0    192.168.1.1    192.168.1.100      25
      127.0.0.0    255.0.0.0        On-link       127.0.0.1     306
    192.168.1.0  255.255.255.0      On-link    192.168.1.100     281
  192.168.1.100  255.255.255.255    On-link    192.168.1.100     281
  192.168.1.255  255.255.255.255    On-link    192.168.1.100     281
        224.0.0.0    240.0.0.0      On-link       127.0.0.1     306
  255.255.255.255  255.255.255.255  On-link       127.0.0.1     306
```

| Column | Meaning |
|---|---|
| Network Destination | Target network or host |
| Netmask | Subnet mask for the destination |
| Gateway | Next-hop IP (`On-link` = directly connected) |
| Interface | Local IP used to reach the gateway |
| Metric | Route cost (lower = more preferred) |

## Important Standard Routes

| Destination | Meaning |
|---|---|
| 0.0.0.0 / 0.0.0.0 | Default route (all traffic) |
| 127.0.0.0 / 255.0.0.0 | Loopback |
| 224.0.0.0 / 240.0.0.0 | Multicast range |
| 255.255.255.255 | Limited broadcast |

## Filtering for a Specific Route

```cmd
:: Show only the default route
route print | findstr "0.0.0.0"

:: Show routes for a specific subnet
route print | findstr "10.0.0"

:: Show persistent routes only
route print | findstr /i "persistent"
```

## Viewing Persistent Routes

Persistent routes are shown in a separate section at the bottom:

```
Persistent Routes:
  Network Address          Netmask  Gateway Address  Metric
         10.0.0.0        255.0.0.0      192.168.1.254       1
```

## PowerShell Alternative

```powershell
# Show IPv4 routing table with full details
Get-NetRoute -AddressFamily IPv4 | Sort-Object RouteMetric | Format-Table

# Show default routes only
Get-NetRoute -DestinationPrefix "0.0.0.0/0"
```

## Conclusion

`route print -4` shows the full IPv4 routing table. Focus on the default route (0.0.0.0) for gateway issues, look at connected-network routes (On-link) to confirm adapter addresses, and check the Persistent Routes section to verify permanent static routes are in place.
