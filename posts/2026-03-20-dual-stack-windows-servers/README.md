# How to Configure Dual-Stack on Windows Servers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, IPv4, Dual-Stack, Windows, PowerShell

Description: Learn how to configure dual-stack IPv4 and IPv6 networking on Windows Server using PowerShell, netsh, and GUI tools for static and dynamic addressing.

## Overview

Windows Server supports dual-stack natively. Both IPv4 and IPv6 stacks are installed and active by default. Configuring dual-stack means assigning static IPv6 addresses (or enabling SLAAC/DHCPv6), setting a default gateway, and verifying routing and DNS resolution for both families.

## Check Current Configuration

```powershell
# Show all IP addresses

Get-NetIPAddress | Select-Object InterfaceAlias, AddressFamily, IPAddress, PrefixLength

# Show routes for both families
Get-NetRoute | Select-Object DestinationPrefix, NextHop, RouteMetric | Where-Object DestinationPrefix -match "::"

# Show DNS servers
Get-DnsClientServerAddress | Select-Object InterfaceAlias, AddressFamily, ServerAddresses

# Test connectivity
Test-NetConnection -ComputerName 8.8.8.8           # IPv4
Test-NetConnection -ComputerName 2001:4860:4860::8888  # IPv6
```

## Assign Static Dual-Stack Addresses (PowerShell)

```powershell
# Get interface index
$iface = Get-NetAdapter -Name "Ethernet" | Select-Object -ExpandProperty ifIndex

# Remove existing IPv6 address if any
Remove-NetIPAddress -InterfaceIndex $iface -AddressFamily IPv6 -Confirm:$false -ErrorAction SilentlyContinue

# Add IPv4 address (if not already set)
New-NetIPAddress -InterfaceIndex $iface `
    -AddressFamily IPv4 `
    -IPAddress "192.0.2.10" `
    -PrefixLength 24 `
    -DefaultGateway "192.0.2.1"

# Add IPv6 address
New-NetIPAddress -InterfaceIndex $iface `
    -AddressFamily IPv6 `
    -IPAddress "2001:db8::10" `
    -PrefixLength 64 `
    -DefaultGateway "2001:db8::1"

# Set DNS servers (both IPv4 and IPv6 resolvers)
Set-DnsClientServerAddress -InterfaceIndex $iface `
    -ServerAddresses "192.0.2.53", "2001:db8::53", "8.8.8.8", "2001:4860:4860::8888"
```

## Netsh Method (Server Core / Legacy Scripts)

```cmd
:: Set IPv4
netsh interface ipv4 set address "Ethernet" static 192.0.2.10 255.255.255.0 192.0.2.1

:: Set IPv6
netsh interface ipv6 set address "Ethernet" 2001:db8::10/64

:: Set IPv6 default gateway
netsh interface ipv6 add route ::/0 "Ethernet" 2001:db8::1

:: Set DNS
netsh interface ipv4 set dns "Ethernet" static 192.0.2.53
netsh interface ipv6 set dns "Ethernet" static 2001:db8::53
```

## Enable SLAAC and DHCPv6 (Dynamic)

```powershell
# Allow Router Advertisements (for SLAAC default route and address)
Set-NetIPv6Protocol -RouterDiscovery Enabled

# Enable DHCPv6 for interface (complement to SLAAC)
Set-NetIPInterface -InterfaceIndex $iface -AddressFamily IPv6 -Dhcp Enabled

# Check RA-configured addresses
Get-NetIPAddress -InterfaceIndex $iface -AddressFamily IPv6 | Where-Object PrefixOrigin -eq "RouterAdvertisement"
```

## Verify Dual-Stack Configuration

```powershell
# Full interface configuration
Get-NetIPConfiguration -InterfaceAlias "Ethernet"

# Sample output:
# InterfaceAlias       : Ethernet
# IPv4Address          : 192.0.2.10
# IPv4DefaultGateway   : 192.0.2.1
# IPv6Address          : 2001:db8::10
# IPv6DefaultGateway   : 2001:db8::1
# DNSServer            : 192.0.2.53, 2001:db8::53

# Resolve hostname - should show A and AAAA
Resolve-DnsName example.com | Select-Object Name, Type, IPAddress

# Traceroute via IPv6
tracert -6 2001:4860:4860::8888

# Traceroute via IPv4
tracert -4 8.8.8.8
```

## Address Preference (RFC 6724)

Windows implements RFC 6724 address selection. IPv6 global unicast is preferred:

```powershell
# View prefix policy table
netsh interface ipv6 show prefixpolicies

# Sample output:
# Precedence  Label  Prefix
#     50       0      ::1/128          (loopback)
#     40       1      ::/0             (global IPv6) ← preferred
#     35       4      ::ffff:0:0/96    (IPv4-mapped)
#     30       2      2002::/16        (6to4)
#      5      5      2001::/32        (Teredo)

# Force IPv4 preference (not recommended - workaround for broken IPv6)
netsh interface ipv6 set prefixpolicy ::ffff:0:0/96 100 4
```

## Disable Unnecessary IPv6 Features

```powershell
# Disable 6to4 (deprecated)
Set-Net6to4Configuration -State Disabled

# Disable Teredo (tunneling through NAT)
Set-NetTeredoConfiguration -Type Disabled

# Disable ISATAP
Set-NetIsatapConfiguration -State Disabled

# Verify all tunneling disabled
Get-Net6to4Configuration
Get-NetTeredoConfiguration
Get-NetIsatapConfiguration
```

## Firewall Rules for Dual-Stack

After configuring addresses, verify firewall covers both:

```powershell
# Check if firewall allows IPv6 SSH (if needed)
Get-NetFirewallRule | Where-Object {
    $_.DisplayName -like "*SSH*" -or $_.DisplayName -like "*Remote*"
} | Select-Object DisplayName, Enabled, Action

# Add rule for IPv6 management
New-NetFirewallRule `
    -DisplayName "Allow SSH IPv6 Management" `
    -Direction Inbound `
    -Protocol TCP `
    -LocalPort 22 `
    -AddressFamily IPv6 `
    -RemoteAddress "fd00:mgmt::/48" `
    -Action Allow `
    -Enabled True
```

## Server Application Dual-Stack Binding

Most Windows services bind to all addresses by default. Verify:

```cmd
:: Check what ports are listening on IPv6
netstat -an | findstr "LISTENING" | findstr ":::"

:: IIS: ensure binding covers IPv6
:: In IIS Manager → Site → Bindings → Edit
:: IP Address: All Unassigned  → handles both IPv4 and IPv6
```

## Summary

Windows Server dual-stack uses `New-NetIPAddress` for static addressing of both families, with `Set-DnsClientServerAddress` for DNS. Disable tunneling mechanisms (6to4, Teredo, ISATAP) to avoid accidental IPv6 bypass paths. Windows prefers IPv6 by default per RFC 6724 - override only if IPv6 is broken, not as a permanent configuration. Ensure firewall rules cover both address families and verify applications listen on `:::port` for IPv6 alongside `0.0.0.0:port` for IPv4.
