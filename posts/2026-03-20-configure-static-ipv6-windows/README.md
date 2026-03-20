# How to Configure Static IPv6 Addresses on Windows

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Windows, Static Address, PowerShell, Netsh

Description: Learn how to assign static IPv6 addresses on Windows using PowerShell, netsh, and the GUI, including setting a gateway and DNS servers for full IPv6 connectivity.

## Configure Static IPv6 via PowerShell

```powershell
# Get the interface index or alias

Get-NetAdapter | Select-Object Name, InterfaceIndex, Status

# Add a static IPv6 address
New-NetIPAddress `
    -InterfaceAlias "Ethernet" `
    -IPAddress "2001:db8::10" `
    -PrefixLength 64 `
    -DefaultGateway "2001:db8::1"

# Verify the address
Get-NetIPAddress -InterfaceAlias "Ethernet" -AddressFamily IPv6
```

## Configure Static IPv6 via netsh

```cmd
:: Show current IPv6 configuration
netsh interface ipv6 show addresses

:: Add a static IPv6 address
netsh interface ipv6 add address "Ethernet" 2001:db8::10/64

:: Add default gateway
netsh interface ipv6 add route ::/0 "Ethernet" 2001:db8::1

:: Add DNS server
netsh interface ipv6 add dnsserver "Ethernet" 2001:4860:4860::8888 index=1
netsh interface ipv6 add dnsserver "Ethernet" 2001:4860:4860::8844 index=2

:: Verify
netsh interface ipv6 show addresses
netsh interface ipv6 show routes
```

## Configure Static IPv6 via GUI

```sql
Steps:
1. Open ncpa.cpl (Win + R → ncpa.cpl)

2. Right-click adapter → Properties

3. Select "Internet Protocol Version 6 (TCP/IPv6)" → Properties

4. Select "Use the following IPv6 address"

5. Enter:
   - IPv6 address: 2001:db8::10
   - Subnet prefix length: 64
   - Default gateway: 2001:db8::1

6. Under DNS:
   - Preferred DNS: 2001:4860:4860::8888
   - Alternate DNS: 2001:4860:4860::8844

7. Click OK
```

## Remove Static IPv6 Address

```powershell
# Remove a specific IPv6 address
Remove-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress "2001:db8::10" -Confirm:$false

# Remove the default route
Remove-NetRoute -InterfaceAlias "Ethernet" -DestinationPrefix "::/0" -Confirm:$false

# Remove all manually-configured IPv6 addresses
Get-NetIPAddress -InterfaceAlias "Ethernet" -AddressFamily IPv6 |
    Where-Object {$_.PrefixOrigin -eq "Manual"} |
    Remove-NetIPAddress -Confirm:$false
```

## Configure Multiple IPv6 Addresses

```powershell
# Windows supports multiple IPv6 addresses per interface

# Add primary address
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress "2001:db8::10" -PrefixLength 64

# Add secondary address
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress "2001:db8::20" -PrefixLength 64

# Add a ULA address
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress "fd00:db8::10" -PrefixLength 48

# View all IPv6 addresses
Get-NetIPAddress -InterfaceAlias "Ethernet" -AddressFamily IPv6
```

## Verify and Test Static IPv6

```powershell
# View full IPv6 configuration
Get-NetIPConfiguration -InterfaceAlias "Ethernet"

# Show IPv6 routing table
Get-NetRoute -AddressFamily IPv6

# Test connectivity to gateway
Test-NetConnection -ComputerName "2001:db8::1"

# Test external IPv6 connectivity
Test-NetConnection -ComputerName "2001:4860:4860::8888" -Port 443

# Ping with IPv6
ping -6 2001:4860:4860::8888
```

## Summary

Assign static IPv6 on Windows with `New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress "2001:db8::10" -PrefixLength 64 -DefaultGateway "2001:db8::1"` and set DNS with `Set-DnsClientServerAddress`. Use `netsh interface ipv6 add address` for a command-line alternative. The GUI method via **ncpa.cpl → IPv6 Properties** works without PowerShell. Verify with `Get-NetIPConfiguration` and `Test-NetConnection`.
