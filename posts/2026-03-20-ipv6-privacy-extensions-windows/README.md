# How to Configure IPv6 Privacy Extensions on Windows - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Privacy Extensions, Window, PowerShell, Netsh, Security

Description: A guide to enabling and configuring IPv6 privacy extensions on Windows 10 and Windows 11 using netsh, PowerShell, and Group Policy, to ensure temporary IPv6 addresses are used for outbound...

Windows enables IPv6 privacy extensions by default, but the configuration can be customized or may have been changed by administrators. This guide explains how to verify, enable, and manage IPv6 privacy extensions on Windows systems.

## Checking Current Privacy Extension State

```powershell
# Check IPv6 privacy address settings via PowerShell

Get-NetIPv6Protocol | Select-Object UseTemporaryAddresses, MaxTemporaryDadAttempts, RegenerateTime, MaxRandomTime, PrefixDelegationEnabled

# Check per-interface
Get-NetAdapter | ForEach-Object {
  $adapter = $_
  $ipv6 = Get-NetIPv6Protocol -InterfaceAlias $_.Name -ErrorAction SilentlyContinue
  if ($ipv6) {
    [PSCustomObject]@{
      Interface = $adapter.Name
      UseTemporaryAddresses = $ipv6.UseTemporaryAddresses
      RegenerateTime = $ipv6.RegenerateTime
    }
  }
}

# Check via netsh
netsh interface ipv6 show privacy
```

## Enabling Privacy Extensions via PowerShell

```powershell
# Enable privacy extensions globally (affects all interfaces)
Set-NetIPv6Protocol -UseTemporaryAddresses Enabled

# Enable for a specific interface
Set-NetIPv6Protocol -InterfaceAlias "Wi-Fi" -UseTemporaryAddresses Enabled
Set-NetIPv6Protocol -InterfaceAlias "Ethernet" -UseTemporaryAddresses Enabled

# Configure regeneration time (how often temp address is regenerated)
# Default: 7200 seconds (2 hours). Lower = more frequent rotation
Set-NetIPv6Protocol -UseTemporaryAddresses Enabled -RegenerateTime 3600

# Check current addresses (temporary ones show "Temporary" type)
Get-NetIPAddress -AddressFamily IPv6 | Where-Object { $_.PrefixOrigin -eq "RouterAdvertisement" } |
  Select-Object InterfaceAlias, IPAddress, SuffixOrigin, AddressState, ValidLifetime, PreferredLifetime
```

## Using netsh for Legacy Management

```cmd
REM Check privacy state
netsh interface ipv6 show privacy

REM Enable privacy extensions
netsh interface ipv6 set privacy state=enabled

REM Enable for specific interface
netsh interface ipv6 set privacy name="Wi-Fi" state=enabled

REM View assigned IPv6 addresses (temporary have a * prefix in some outputs)
netsh interface ipv6 show addresses

REM Check which address is used for outbound connections
netsh interface ipv6 show route
```

## Verifying Temporary Addresses Are Used

```powershell
# Check all IPv6 addresses on the system
Get-NetIPAddress -AddressFamily IPv6 |
  Where-Object { $_.SuffixOrigin -eq "Random" } |
  Select-Object InterfaceAlias, IPAddress, ValidLifetime, PreferredLifetime

# The "Random" SuffixOrigin indicates privacy extension addresses

# Test which address is used for outbound connections
# (Check via a website that shows your IPv6)
Invoke-WebRequest -Uri "https://ipv6.icanhazip.com" -UseBasicParsing |
  Select-Object -ExpandProperty Content
# Should show a random-looking temporary address
```

## Windows Address Types

```powershell
# Windows IPv6 address SuffixOrigin values:
# ManuallyConfigured = static
# WellKnown = loopback/link-local prefix
# OriginDhcp = DHCPv6 assigned
# LinkLayerAddress = EUI-64 derived (old, less private)
# Random = privacy extension temporary address (desired)

# Count addresses by type
Get-NetIPAddress -AddressFamily IPv6 -InterfaceAlias "Wi-Fi" |
  Group-Object SuffixOrigin | Select-Object Name, Count
```

## Group Policy for Enterprise Management

```powershell
# Group Policy path:
# Computer Configuration > Administrative Templates >
# Network > TCPIP Settings > IPv6 Transition Technologies

# Via registry (for scripted deployment)
# HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip6\Parameters
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip6\Parameters" `
  -Name "UseTemporaryAddresses" -Value 1 -Type DWord

# Value meanings:
# 0 = disabled
# 1 = enabled (Windows default)
# 2 = always prefer temporary

# Also configure maximum lifetime via registry
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip6\Parameters" `
  -Name "MaxTemporaryLifetime" -Value 604800 -Type DWord   # 7 days in seconds

Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip6\Parameters" `
  -Name "RegenerateTime" -Value 3600 -Type DWord   # Regenerate every hour
```

## Disabling Privacy Extensions for Servers

Server systems that need stable IPv6 addresses should disable privacy extensions:

```powershell
# Disable privacy extensions on a Windows Server
Set-NetIPv6Protocol -UseTemporaryAddresses Disabled

# Assign a static IPv6 address instead
New-NetIPAddress -InterfaceAlias "Ethernet" `
  -IPAddress "2001:db8::server1" `
  -PrefixLength 64 `
  -DefaultGateway "2001:db8::1"

# Verify only the static address is present (no temporary addresses)
Get-NetIPAddress -InterfaceAlias "Ethernet" -AddressFamily IPv6 |
  Select-Object IPAddress, SuffixOrigin, PrefixOrigin
```

## Troubleshooting Privacy Extensions on Windows

```powershell
# If temporary addresses aren't being assigned, check RA receipt
# Look for Router Advertisement received events
Get-NetIPv6Protocol | Select-Object RouterDiscovery

# Verify the router is advertising the M and A flags
# (A flag = Autonomous address configuration = SLAAC with privacy)
netsh interface ipv6 show interfaces

# Reset IPv6 stack if privacy extensions stopped working
netsh interface ipv6 reset
# Then re-enable privacy extensions
Set-NetIPv6Protocol -UseTemporaryAddresses Enabled

# Check Windows Firewall isn't blocking IPv6 Neighbor Discovery
Get-NetFirewallRule | Where-Object { $_.Enabled -eq "True" -and $_.DisplayName -like "*ICMPv6*" }
```

Windows enables IPv6 privacy extensions by default, generating temporary addresses that rotate approximately every 24 hours. For enterprise environments, use Group Policy or registry settings to enforce privacy extension configuration across managed systems. Server systems that require stable addresses should explicitly disable privacy extensions and use statically configured IPv6 addresses instead.
