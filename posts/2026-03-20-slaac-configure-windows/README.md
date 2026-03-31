# How to Configure SLAAC on Windows

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: SLAAC, Window, IPv6, Address Autoconfiguration, Netsh, PowerShell

Description: Configure and verify IPv6 SLAAC on Windows using netsh and PowerShell commands, including managing temporary addresses and privacy extensions.

## Introduction

Windows enables IPv6 SLAAC by default for all network interfaces. The Windows IPv6 stack processes Router Advertisements, generates addresses from prefixes (using privacy extensions by default since Vista), and sets up default routes automatically. This guide covers verifying, configuring, and troubleshooting SLAAC on Windows using `netsh` and PowerShell.

## Verifying SLAAC Status on Windows

```powershell
# Show all IPv6 addresses including SLAAC-generated

Get-NetIPAddress -AddressFamily IPv6

# Example output:
# IPAddress         : 2001:db8::a3f2:1b8c:9d4e:7f05
# InterfaceAlias    : Ethernet
# AddressFamily     : IPv6
# PrefixLength      : 64
# PrefixOrigin      : RouterAdvertisement   ← SLAAC
# SuffixOrigin      : Random               ← Privacy extensions
# AddressState      : Preferred
# ValidLifetime     : Infinite              ← Actually from RA
#
# IPAddress         : fe80::3f91:28ac:e654:102b
# InterfaceAlias    : Ethernet
# PrefixOrigin      : WellKnown
# SuffixOrigin      : Random
# AddressState      : Preferred

# Show SLAAC addresses only (RouterAdvertisement origin)
Get-NetIPAddress -AddressFamily IPv6 |
    Where-Object { $_.PrefixOrigin -eq "RouterAdvertisement" } |
    Format-Table IPAddress, InterfaceAlias, AddressState, ValidLifetime
```

## Configuring SLAAC with netsh

```cmd
REM Show current IPv6 interface configuration
netsh interface ipv6 show interface "Ethernet"

REM Show all IPv6 addresses
netsh interface ipv6 show addresses

REM Enable SLAAC (Router Discovery)
netsh interface ipv6 set interface "Ethernet" routerdiscovery=enabled

REM Disable SLAAC (use static addresses only)
netsh interface ipv6 set interface "Ethernet" routerdiscovery=disabled

REM Show Router Advertisement information
netsh interface ipv6 show routes
REM Shows routes including those from RA

REM Force sending a Router Solicitation (refresh SLAAC)
netsh interface ipv6 set interface "Ethernet" routerdiscovery=enabled
REM Disabling and re-enabling triggers RS
```

## Managing Privacy Extensions on Windows

```powershell
# Check privacy extension status
netsh interface ipv6 show privacy
# State      : enabled      ← Privacy extensions on
# MaxDadAttempts : 1
# MaxPreferredLifetime : 1d
# MaxValidLifetime : 7d
# RegenerateTime : 5s

# Disable privacy extensions (use stable EUI-64 addresses)
netsh interface ipv6 set privacy state=disabled

# Enable privacy extensions
netsh interface ipv6 set privacy state=enabled

# Preferred lifetime for temporary addresses (default: 1 day)
netsh interface ipv6 set privacy maxpreferredlifetime=12h

# Valid lifetime for temporary addresses (default: 7 days)
netsh interface ipv6 set privacy maxvalidlifetime=1d

# Check current temporary address details
Get-NetIPAddress -AddressFamily IPv6 |
    Where-Object { $_.SuffixOrigin -eq "Random" } |
    Format-Table IPAddress, ValidLifetime, PreferredLifetime
```

## Static vs SLAAC on Windows

```powershell
# Configure static IPv6 address (disables SLAAC for that address)
# SLAAC continues for other prefixes received from RA
New-NetIPAddress -InterfaceAlias "Ethernet" `
    -IPAddress "2001:db8::10" `
    -PrefixLength 64 `
    -DefaultGateway "fe80::1"

# Remove static address
Remove-NetIPAddress -IPAddress "2001:db8::10" -Confirm:$false

# Check if SLAAC is disabled for interface
Get-NetIPInterface -InterfaceAlias "Ethernet" -AddressFamily IPv6 |
    Select RouterDiscovery, ManagedAddressConfiguration,
           OtherStatefulConfiguration
# RouterDiscovery               : Enabled
# ManagedAddressConfiguration   : Disabled   ← M flag (stateful DHCPv6)
# OtherStatefulConfiguration    : Disabled   ← O flag (stateless DHCPv6)

# Force SLAAC refresh
# (Disable and re-enable the interface)
Disable-NetAdapter -Name "Ethernet" -Confirm:$false
Start-Sleep -Seconds 2
Enable-NetAdapter -Name "Ethernet"
```

## Verifying Routes from SLAAC

```powershell
# Show IPv6 routes learned via SLAAC (RouterAdvertisement protocol)
Get-NetRoute -AddressFamily IPv6 |
    Where-Object { $_.Protocol -eq "RouterAdvertisement" } |
    Format-Table DestinationPrefix, NextHop, InterfaceAlias, RouteMetric

# Example output:
# DestinationPrefix  NextHop       InterfaceAlias   RouteMetric
# ::/0               fe80::1       Ethernet         256   ← default route
# 2001:db8::/64      ::            Ethernet         256   ← on-link route

# Show all IPv6 routes
Get-NetRoute -AddressFamily IPv6 | Format-Table -AutoSize

# netsh equivalent:
netsh interface ipv6 show routes
```

## Troubleshooting SLAAC on Windows

```powershell
# Problem: No SLAAC address received

# Check 1: Is IPv6 enabled on the adapter?
Get-NetAdapterBinding -Name "Ethernet" -ComponentID ms_tcpip6
# If Enabled is False: IPv6 is disabled
# Fix: Enable-NetAdapterBinding -Name "Ethernet" -ComponentID ms_tcpip6

# Check 2: Is Router Discovery enabled?
Get-NetIPInterface -InterfaceAlias "Ethernet" -AddressFamily IPv6 |
    Select RouterDiscovery
# Expected: Enabled

# Check 3: Capture RA messages (requires Wireshark or pktmon)
# Using pktmon (Windows 10+):
pktmon filter add -p icmpv6
pktmon start --capture
# Wait a few seconds, then stop:
pktmon stop
pktmon etl2txt PktMon.etl
# Look for ICMPv6 Type 134 (Router Advertisement)

# Check 4: Firewall blocking RA?
# ICMPv6 Type 134 must be allowed inbound
netsh advfirewall firewall show rule name="Core Networking - Router Advertisement (ICMPv6-In)"

# Check 5: Check event log for IPv6 issues
Get-WinEvent -LogName "System" |
    Where-Object { $_.Message -like "*IPv6*" } |
    Select -First 10
```

## Windows Server SLAAC Configuration

```powershell
# On Windows Server: verify SLAAC is enabled
# Windows Server may disable IPv6 components

# Check IPv6 preference
Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip6\Parameters" |
    Select DisabledComponents
# 0x00 = all IPv6 components enabled (SLAAC works)
# 0xFF = all IPv6 components disabled

# Enable all IPv6 components (if disabled)
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip6\Parameters" `
    -Name "DisabledComponents" -Value 0x00 -Type DWord
Restart-Computer

# Enable Router Discovery on specific interface
Set-NetIPInterface -InterfaceAlias "Ethernet" `
    -AddressFamily IPv6 `
    -RouterDiscovery Enabled
```

## Conclusion

Windows enables SLAAC by default with privacy extensions, generating random temporary addresses from RA-advertised prefixes. Use `Get-NetIPAddress` to view SLAAC-assigned addresses (`PrefixOrigin: RouterAdvertisement`). Control privacy extensions with `netsh interface ipv6 set privacy state=enabled/disabled`. For troubleshooting, verify Router Discovery is enabled with `Get-NetIPInterface` and capture RA messages with pktmon or Wireshark. Windows Server may require explicit IPv6 enablement via the `DisabledComponents` registry key.
