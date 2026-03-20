# How to Disable IPv6 Transition Technologies on Windows

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Windows, Teredo, ISATAP, 6to4, Security

Description: Learn how to disable Teredo, ISATAP, and 6to4 IPv6 transition technologies on Windows to reduce attack surface, prevent unexpected tunnel traffic, and avoid connectivity issues on native dual-stack networks.

## Why Disable Transition Technologies?

On networks with native IPv6, legacy transition tunnels can:
- Create unexpected outbound traffic bypassing firewalls
- Introduce security exposure (Teredo bypasses some firewalls)
- Cause connectivity issues when native IPv6 is available
- Interfere with network monitoring and logging

## Disable Teredo

```cmd
:: Disable Teredo client
netsh interface teredo set state type=disabled

:: Verify
netsh interface teredo show state
:: Type should show: disabled
```

```powershell
# Disable via registry (persistent across reboots)
Set-ItemProperty `
    -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip6\Parameters" `
    -Name DisabledComponents `
    -Value 0x08 `
    -Type DWord
# Bit 3 (0x08) disables preferred IPv6 source; bit 6 (0x40) disables Teredo
# Use 0x40 for Teredo-only disable
```

## Disable 6to4

```cmd
:: Disable 6to4
netsh interface 6to4 set state disabled

:: Verify
netsh interface 6to4 show state
:: State should show: disabled
```

## Disable ISATAP

```cmd
:: Disable ISATAP
netsh interface isatap set state disabled

:: Verify
netsh interface isatap show state
:: State should show: disabled
```

## Disable All Three with One Script

```powershell
# Comprehensive script to disable all IPv6 transition technologies

Write-Host "Disabling IPv6 transition technologies..."

# Disable Teredo
netsh interface teredo set state type=disabled
Write-Host "Teredo: disabled"

# Disable 6to4
netsh interface 6to4 set state disabled
Write-Host "6to4: disabled"

# Disable ISATAP
netsh interface isatap set state disabled
Write-Host "ISATAP: disabled"

# Also disable via registry (bits 4, 5, 6 for ISATAP, 6to4, Teredo)
$regPath = "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip6\Parameters"
if (-not (Test-Path $regPath)) {
    New-Item -Path $regPath -Force | Out-Null
}
# 0x70 = ISATAP(0x10) + 6to4(0x20) + Teredo(0x40)
Set-ItemProperty -Path $regPath -Name DisabledComponents -Value 0x70 -Type DWord
Write-Host "Registry DisabledComponents set to 0x70 (tunnels disabled)"

Write-Host ""
Write-Host "Verification:"
netsh interface teredo show state | Select-String "Type"
netsh interface 6to4 show state | Select-String "State"
netsh interface isatap show state | Select-String "State"
```

## Using Group Policy to Disable Across Enterprise

```
Group Policy path:
Computer Configuration →
  Administrative Templates →
    Network →
      TCPIP Settings →
        IPv6 Transition Technologies

Settings to configure:
- Set Teredo State = Disabled
- Set 6to4 State = Disabled
- Set ISATAP State = Disabled
```

## Verify All Tunnels are Disabled

```powershell
# Check all tunnel adapters
Get-NetAdapter | Where-Object {
    $_.InterfaceDescription -match "Tunnel|Teredo|ISATAP|6to4|Microsoft 6to4"
}

# Should show no active tunnel adapters

# Check state of each
Write-Host "=== Teredo ==="
netsh interface teredo show state

Write-Host "=== 6to4 ==="
netsh interface 6to4 show state

Write-Host "=== ISATAP ==="
netsh interface isatap show state

# Check registry
(Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip6\Parameters" `
    -Name DisabledComponents -ErrorAction SilentlyContinue).DisabledComponents
```

## Summary

Disable legacy IPv6 transition technologies on Windows with `netsh interface teredo/6to4/isatap set state disabled`. For persistent disable across reboots, set the `DisabledComponents` registry value to `0x70` (bits for ISATAP, 6to4, and Teredo). Use Group Policy for enterprise-wide deployment. These tunnels are unnecessary on native dual-stack networks and should be disabled to reduce attack surface and prevent unexpected traffic patterns.
