# How to Create DHCP Reservations for Specific IPv4 Addresses on Windows Server

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Window, DHCP, IPv4, Windows Server, Reservations, PowerShell

Description: Create DHCP reservations on Windows Server to ensure specific devices always receive the same IPv4 address based on their MAC address, using both PowerShell and the DHCP Manager GUI.

## Introduction

A DHCP reservation ties a specific IPv4 address to a MAC address. The DHCP server always assigns the reserved address to the matching device, giving it a "pseudo-static" IP while still managing it through DHCP - eliminating the need to configure static IPs on the device itself.

## Creating a Reservation via PowerShell

```powershell
# Create a reservation for a printer with MAC 00-1A-2B-3C-4D-5E

Add-DhcpServerv4Reservation `
    -ScopeId 192.168.1.0 `
    -IPAddress 192.168.1.50 `
    -ClientId "00-1A-2B-3C-4D-5E" `
    -Name "Office-Printer-01" `
    -Description "HP LaserJet - Conference Room"
```

## Creating Multiple Reservations from a CSV

```powershell
# reservations.csv format:
# IPAddress,MAC,Name,Description
# 192.168.1.50,00-1A-2B-3C-4D-5E,Printer01,HP LaserJet
# 192.168.1.51,AA-BB-CC-DD-EE-FF,Camera01,IP Camera Lobby
# 192.168.1.52,11-22-33-44-55-66,AP01,Wireless AP Floor 1

Import-Csv -Path "C:\reservations.csv" | ForEach-Object {
    Add-DhcpServerv4Reservation `
        -ScopeId 192.168.1.0 `
        -IPAddress $_.IPAddress `
        -ClientId $_.MAC `
        -Name $_.Name `
        -Description $_.Description
    Write-Host "Reserved $($_.IPAddress) for $($_.Name)"
}
```

## Viewing Existing Reservations

```powershell
# List all reservations in a scope
Get-DhcpServerv4Reservation -ScopeId 192.168.1.0

# Find reservation for a specific MAC
Get-DhcpServerv4Reservation -ScopeId 192.168.1.0 | Where-Object {$_.ClientId -eq "00-1A-2B-3C-4D-5E"}
```

## Removing a Reservation

```powershell
# Remove by IP address
Remove-DhcpServerv4Reservation -ScopeId 192.168.1.0 -IPAddress 192.168.1.50

# Remove multiple at once
@("192.168.1.50", "192.168.1.51") | ForEach-Object {
    Remove-DhcpServerv4Reservation -ScopeId 192.168.1.0 -IPAddress $_
}
```

## GUI: Creating a Reservation in DHCP Manager

1. Open **DHCP Manager**
2. Navigate to the scope → **Reservations**
3. Right-click → **New Reservation**
4. Enter:
   - **Reservation name**: `Printer01`
   - **IP address**: `192.168.1.50`
   - **MAC address**: `001A2B3C4D5E` (no hyphens in GUI)
   - **Description**: optional
5. Click **Add**

## Forcing the Client to Use the Reserved IP

After creating the reservation, the client needs to renew its lease:

```cmd
:: On the client machine
ipconfig /release
ipconfig /renew
```

The DHCP server will now offer the reserved IP.

## Finding a Device's MAC Address

```cmd
:: On Windows client - find MAC for creating a reservation
getmac /v
ipconfig /all | findstr "Physical"

:: On Linux client
ip link show
```

## Conclusion

DHCP reservations combine the simplicity of DHCP (no manual IP configuration on devices) with the predictability of static addresses. Use PowerShell's `Add-DhcpServerv4Reservation` for bulk creation from a CSV, and `Get-DhcpServerv4Reservation` to audit the current state.
