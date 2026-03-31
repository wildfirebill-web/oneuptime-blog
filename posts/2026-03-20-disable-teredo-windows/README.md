# How to Disable Teredo on Windows to Prevent IPv6 Leaks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Teredo, Window, Security, NAT Bypass

Description: Learn how to permanently disable Teredo on Windows to prevent IPv6 connectivity leaks through NAT, including netsh commands, PowerShell, Group Policy, and registry methods.

## Overview

Teredo is enabled or dormant by default on Windows Vista through Windows 10. When active, it creates IPv6 connectivity through NAT by tunneling over UDP port 3544, bypassing IPv4 firewalls that don't inspect UDP payloads for IPv6 content. Disabling Teredo is a security hardening step recommended by NIST, NSA, and CIS benchmarks for enterprise environments.

## Verify Teredo Is Active

```cmd
netsh interface teredo show state
```

Teredo is a threat if state is `client` or `probe`:
```text
Type              : client            ← ACTIVE - needs to be disabled
Server Name       : teredo.ipv6.microsoft.com
Mapped Address    : 203.0.113.50:32000
Network           : unmanaged
NAT               : cone
NAT Special Behaviour: UPNP: No, PortPreserving: Yes
Local mapping     : 192.168.1.10:32000
Client Refresh    : No
```

If state is `dormant`, Teredo is installed but inactive (native IPv6 present). Still disable it:
```text
Type              : dormant           ← inactive but can become active
```

## Disable Teredo: netsh Method

```cmd
:: Disable Teredo
netsh interface teredo set state disabled

:: Verify
netsh interface teredo show state
:: Expected: Type : disabled
```

## Disable Teredo: PowerShell Method

```powershell
# Disable Teredo

Set-NetTeredoConfiguration -Type Disabled

# Verify
Get-NetTeredoConfiguration | Select-Object Type
# Type: Disabled
```

## Disable All IPv6 Transition Mechanisms

Best practice: disable all three together:

```powershell
# Disable Teredo, 6to4, and ISATAP
Set-NetTeredoConfiguration -Type Disabled
Set-Net6to4Configuration -State Disabled
Set-NetIsatapConfiguration -State Disabled

# Verify all three
Get-NetTeredoConfiguration | Select-Object Type
Get-Net6to4Configuration | Select-Object State
Get-NetIsatapConfiguration | Select-Object State
```

## Persistent Disable via Registry

Registry method survives some resets and is useful for scripted hardening:

```powershell
# Registry path for IPv6 transition components
$regPath = "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip6\Parameters"

# Bit flags for DisabledComponents:
# 0x01 = disable IPv6 on tunnel interfaces (6to4, ISATAP, Teredo)
# 0x08 = disable Teredo
# 0x20 = disable 6to4
# 0xFF = disable all IPv6 (not recommended unless IPv6 truly not needed)

# Disable only tunneling (keep native IPv6)
$value = 0x01  # Disable all tunnel interfaces

Set-ItemProperty -Path $regPath -Name "DisabledComponents" -Value $value -Type DWord

# To disable Teredo only (bit 0x08):
$current = (Get-ItemProperty -Path $regPath -ErrorAction SilentlyContinue).DisabledComponents
if ($null -eq $current) { $current = 0 }
Set-ItemProperty -Path $regPath -Name "DisabledComponents" -Value ($current -bor 0x08) -Type DWord
```

Note: Registry changes require a reboot to take full effect.

## Group Policy for Enterprise

```text
Computer Configuration →
  Administrative Templates →
    Network →
      TCPIP Settings →
        IPv6 Transition Technologies →

  Set Teredo State
    → Enabled
    → State: Disabled

  Set 6to4 State
    → Enabled
    → State: Disabled

  Set ISATAP State
    → Enabled
    → State: Disabled
```

GPO path (ADMX): `HKLM\Software\Policies\Microsoft\Windows\TCPIP\v6Transition`

## PowerShell Script for Mass Deployment

```powershell
# deploy-disable-teredo.ps1
# Deploy to all domain computers via Invoke-Command or GPO startup script

$ErrorActionPreference = "SilentlyContinue"

# Disable all transition mechanisms
Set-NetTeredoConfiguration -Type Disabled
Set-Net6to4Configuration -State Disabled
Set-NetIsatapConfiguration -State Disabled

# Registry (belt-and-suspenders)
$regPath = "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip6\Parameters"
if (-not (Test-Path $regPath)) {
    New-Item -Path $regPath -Force | Out-Null
}
Set-ItemProperty -Path $regPath -Name "DisabledComponents" -Value 0x01 -Type DWord

# Windows Firewall - block UDP 3544
New-NetFirewallRule -DisplayName "Block Teredo UDP 3544 Outbound" `
    -Direction Outbound -Protocol UDP -RemotePort 3544 `
    -Action Block -Enabled True -Profile Any -ErrorAction SilentlyContinue

New-NetFirewallRule -DisplayName "Block Teredo UDP 3544 Inbound" `
    -Direction Inbound -Protocol UDP -LocalPort 3544 `
    -Action Block -Enabled True -Profile Any -ErrorAction SilentlyContinue

# Log result
$host_result = [pscustomobject]@{
    Computer = $env:COMPUTERNAME
    Timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    TeredoState = (Get-NetTeredoConfiguration).Type
    Status = "Disabled"
}
Write-Output $host_result
```

## Verify After Disablement

```powershell
# Confirm Teredo is gone
Get-NetTeredoConfiguration | Select-Object Type
# Type: Disabled

# Confirm no 2001:0: addresses (Teredo range)
Get-NetIPAddress | Where-Object { $_.IPAddress -match "^2001:0:" }
# Should return nothing

# Confirm no Teredo interface in adapter list
Get-NetAdapter | Where-Object { $_.InterfaceDescription -like "*Teredo*" }
# Should return nothing

# Test: IPv6 connectivity still works via native (if available)
Test-NetConnection -ComputerName "ipv6.google.com" -Port 443
```

## Summary

Disable Teredo on Windows with `netsh interface teredo set state disabled` or `Set-NetTeredoConfiguration -Type Disabled`. For enterprise-wide disablement, use Group Policy (`TCPIP Settings → IPv6 Transition Technologies → Set Teredo State: Disabled`) or a PowerShell deployment script. Set the `DisabledComponents` registry value to `0x01` to disable all tunnel interfaces at the OS level. Also block UDP 3544 at the network firewall as defense-in-depth. Combine with disabling 6to4 and ISATAP to remove all IPv6 tunneling mechanisms.
