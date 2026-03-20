# How to Display All IPv4 Network Adapters with ipconfig

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Windows, Networking, ipconfig, IPv4, Network Diagnostics

Description: Use ipconfig and ipconfig /all to display IPv4 addresses, subnet masks, gateways, and DNS servers for all network adapters on Windows.

## Introduction

`ipconfig` is the standard Windows command for displaying network adapter configuration. It shows assigned IPv4 addresses, subnet masks, default gateways, and DHCP lease information for all adapters in a single output.

## Basic ipconfig

```cmd
ipconfig
```

Shows a summary of IPv4 and IPv6 addresses, subnet masks, and gateways for each adapter:

```
Windows IP Configuration

Ethernet adapter Ethernet:
   Connection-specific DNS Suffix  . : example.com
   IPv4 Address. . . . . . . . . . . : 192.168.1.100
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 192.168.1.1

Wireless LAN adapter Wi-Fi:
   Media State . . . . . . . . . . . : Media disconnected
```

## Detailed Output: ipconfig /all

```cmd
ipconfig /all
```

Adds MAC addresses, DHCP server, lease times, DNS servers, and WINS:

```
Ethernet adapter Ethernet:
   DHCP Enabled. . . . . . . . . . . : No
   IPv4 Address. . . . . . . . . . . : 192.168.1.100(Preferred)
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 192.168.1.1
   DNS Servers . . . . . . . . . . . : 8.8.8.8
                                       1.1.1.1
   Physical Address. . . . . . . . . : 00-1A-2B-3C-4D-5E
```

## Filtering Output for Specific Information

```cmd
:: Show only IPv4 addresses
ipconfig | findstr "IPv4"

:: Show only the gateway
ipconfig | findstr "Default Gateway"

:: Show MAC addresses and IPs from /all
ipconfig /all | findstr /i "Physical\|IPv4\|Gateway"
```

## Displaying a Specific Adapter

```cmd
:: Show full details for just the Ethernet adapter
ipconfig /all | more
:: Navigate to the Ethernet section

:: Or use PowerShell for specific adapter
Get-NetIPAddress -InterfaceAlias "Ethernet" -AddressFamily IPv4
```

## Listing All Adapters Including Disconnected

```cmd
:: All adapters including those with "Media disconnected"
ipconfig /all
```

`Media State: Media disconnected` means the adapter has no active cable or association.

## PowerShell Alternative

```powershell
# Show all IPv4 addresses for all adapters
Get-NetIPAddress -AddressFamily IPv4 | Select-Object InterfaceAlias, IPAddress, PrefixLength, SuffixOrigin

# Show adapter with gateway
Get-NetRoute -AddressFamily IPv4 | Where-Object {$_.NextHop -ne "0.0.0.0"} | Select-Object InterfaceAlias, NextHop, RouteMetric
```

## Exporting ipconfig Output

```cmd
:: Save full network config to a file (useful for documentation/support)
ipconfig /all > C:\network-config.txt
```

## Conclusion

`ipconfig` gives a quick summary; `ipconfig /all` provides the full picture including MAC, DHCP, and DNS details. Use `findstr` to filter specific fields, and PowerShell's `Get-NetIPAddress` for programmatic access to adapter information.
