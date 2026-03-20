# How to Disable IPv6 on Windows via Registry

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Windows, Registry, Network Configuration, Disable IPv6

Description: Learn how to disable IPv6 on Windows by modifying the DisabledComponents registry key, including the different bitmask values that control which IPv6 components are disabled.

## The DisabledComponents Registry Key

Windows controls IPv6 behavior through the `DisabledComponents` DWORD value in the registry. Each bit controls a different aspect of IPv6:

```text
Registry path:
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\Tcpip6\Parameters

Value: DisabledComponents (DWORD)
```

## DisabledComponents Bitmask Values

| Bit | Value | Effect |
|-----|-------|--------|
| 0 | 0x01 | Disable IPv6 on all non-loopback tunnel interfaces |
| 1 | 0x02 | Disable IPv6 on all non-loopback interfaces |
| 2 | 0x04 | Disable IPv6 on loopback interface |
| 3 | 0x08 | Disable preferred IPv6 source addresses |
| 4 | 0x10 | Disable the IPv6 ISATAP tunnel |
| 5 | 0x20 | Disable the IPv6 6to4 tunnel |
| 6 | 0x40 | Disable the IPv6 Teredo tunnel |
| all | 0xFF | Disable all IPv6 components |

## Disabling IPv6 via Registry (Manual)

```text
1. Open Registry Editor: Win + R → regedit
2. Navigate to:
   HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\Tcpip6\Parameters
3. Right-click → New → DWORD (32-bit) Value
4. Name: DisabledComponents
5. Value: 0xFF (to disable all IPv6)
6. Restart the computer
```

## Disabling IPv6 via PowerShell (Registry Method)

```powershell
# Disable all IPv6 components

$registryPath = "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip6\Parameters"

# Check if key exists
if (!(Test-Path $registryPath)) {
    New-Item -Path $registryPath -Force
}

# Set DisabledComponents to 0xFF (disable everything)
Set-ItemProperty -Path $registryPath `
    -Name "DisabledComponents" `
    -Value 0xFF `
    -Type DWord

Write-Host "IPv6 disabled via registry. Restart required."

# Restart to apply
Restart-Computer -Confirm
```

## Disabling Only Tunnel Interfaces (Keep LAN IPv6)

```powershell
# Disable only 6to4, Teredo, ISATAP tunnels but keep native IPv6 on LAN
# Bit 4 (0x10) + Bit 5 (0x20) + Bit 6 (0x40) = 0x70
Set-ItemProperty `
    -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip6\Parameters" `
    -Name "DisabledComponents" `
    -Value 0x70 `
    -Type DWord
```

## Re-enabling IPv6

```powershell
# Set DisabledComponents to 0 (re-enable all)
Set-ItemProperty `
    -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip6\Parameters" `
    -Name "DisabledComponents" `
    -Value 0 `
    -Type DWord

# Or remove the value entirely (same effect as 0)
Remove-ItemProperty `
    -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip6\Parameters" `
    -Name "DisabledComponents" `
    -ErrorAction SilentlyContinue

Restart-Computer
```

## Verifying the Change

```powershell
# Before restart: check registry value
Get-ItemProperty `
    -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip6\Parameters" `
    -Name "DisabledComponents"

# After restart: verify IPv6 is disabled
Get-NetIPAddress -AddressFamily IPv6
# Should show no addresses (or only ::1 on loopback if bit 2 not set)

# Check adapter binding
Get-NetAdapterBinding -ComponentID ms_tcpip6
```

## Summary

Disable IPv6 on Windows by setting `DisabledComponents` DWORD in `HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip6\Parameters`. Use `0xFF` to disable all IPv6, `0x70` to disable only tunnels (6to4, Teredo, ISATAP). A system restart is required for registry changes to take effect. Re-enable by setting the value to `0` or removing it entirely. Microsoft recommends against disabling IPv6 as it may break Windows features that depend on it (HomeGroup, DirectAccess, etc.).
