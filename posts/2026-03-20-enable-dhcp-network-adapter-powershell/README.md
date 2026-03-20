# How to Enable DHCP on a Network Adapter Using PowerShell

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Windows, PowerShell, Networking, DHCP, IPv4, Network Configuration

Description: Enable DHCP on a Windows network adapter using PowerShell cmdlets, remove static IP configuration, and request a new DHCP lease to restore dynamic addressing.

## Introduction

When migrating a Windows machine from static IP to DHCP — such as when moving it from a server rack to an office network — PowerShell provides a clean, scriptable way to enable DHCP and clear static configuration in one sequence.

## Finding the Adapter

```powershell
# List all adapters
Get-NetAdapter | Select-Object Name, InterfaceIndex, Status

# Find the current IP configuration
Get-NetIPAddress -AddressFamily IPv4 | Select-Object InterfaceAlias, IPAddress, PrefixLength, SuffixOrigin
```

## Enabling DHCP (with Static Config Removal)

```powershell
$AdapterName = "Ethernet"
$iface = Get-NetAdapter -Name $AdapterName

# Step 1: Remove existing static IP address(es)
Get-NetIPAddress -InterfaceIndex $iface.InterfaceIndex -AddressFamily IPv4 |
    Remove-NetIPAddress -Confirm:$false -ErrorAction SilentlyContinue

# Step 2: Remove static routes (including default gateway)
Get-NetRoute -InterfaceIndex $iface.InterfaceIndex -AddressFamily IPv4 |
    Remove-NetRoute -Confirm:$false -ErrorAction SilentlyContinue

# Step 3: Enable DHCP
Set-NetIPInterface -InterfaceIndex $iface.InterfaceIndex -Dhcp Enabled

# Step 4: Reset DNS to DHCP-provided
Set-DnsClientServerAddress -InterfaceIndex $iface.InterfaceIndex -ResetServerAddresses

Write-Host "DHCP enabled on $AdapterName"
```

## Requesting a New DHCP Lease Immediately

After enabling DHCP, trigger a lease request:

```powershell
# Bounce the interface to trigger DHCP Discover
Disable-NetAdapter -Name "Ethernet" -Confirm:$false
Enable-NetAdapter  -Name "Ethernet"

# Or use ipconfig
ipconfig /renew "Ethernet"
```

## Checking DHCP Status After Enabling

```powershell
# Confirm DHCP is now active
Get-NetIPInterface -InterfaceAlias "Ethernet" | Select-Object InterfaceAlias, Dhcp

# View the assigned DHCP address
Get-NetIPAddress -InterfaceAlias "Ethernet" -AddressFamily IPv4
```

Expected output:

```
IPAddress      InterfaceAlias PrefixLength SuffixOrigin
---------      -------------- ------------ ------------
192.168.1.105  Ethernet       24           Dhcp
```

`SuffixOrigin: Dhcp` confirms the address was dynamically assigned.

## Netsh Equivalent

```cmd
:: Enable DHCP using netsh (runs in CMD)
netsh interface ipv4 set address name="Ethernet" source=dhcp
netsh interface ipv4 set dns name="Ethernet" source=dhcp
```

## Conclusion

Use `Set-NetIPInterface -Dhcp Enabled` combined with `Remove-NetIPAddress` and `Set-DnsClientServerAddress -ResetServerAddresses` for a complete DHCP transition. Follow with `ipconfig /renew` or an interface bounce to obtain the lease immediately.
