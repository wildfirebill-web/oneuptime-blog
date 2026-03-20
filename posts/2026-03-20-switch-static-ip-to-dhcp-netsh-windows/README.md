# How to Switch a Network Adapter from Static IP to DHCP Using netsh

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Windows, Networking, Netsh, DHCP, IPv4, Network Configuration

Description: Switch a Windows network adapter from a static IPv4 configuration back to DHCP using netsh commands, and reset DNS to automatic assignment as well.

## Introduction

When moving a machine from a static IP environment (such as a data center) to a DHCP-managed network (like an office or cloud subnet), you need to switch the adapter from static to DHCP. `netsh` handles this with a single command.

## Finding the Adapter Name

```cmd
netsh interface show interface
```

Note the exact name (e.g., `Ethernet`, `Ethernet0`, `Local Area Connection`).

## Switching IP Address to DHCP

```cmd
:: Enable DHCP on the "Ethernet" adapter
netsh interface ipv4 set address name="Ethernet" source=dhcp
```

This removes the static address and tells the adapter to request an address from a DHCP server.

## Switching DNS to Automatic (DHCP-provided)

```cmd
:: Set DNS to automatic (obtained via DHCP)
netsh interface ipv4 set dns name="Ethernet" source=dhcp
```

## Releasing and Renewing the DHCP Lease

After switching to DHCP, force an immediate lease request:

```cmd
:: Release the current lease and request a new one
ipconfig /release "Ethernet"
ipconfig /renew "Ethernet"
```

## Verifying the DHCP-Assigned Address

```cmd
ipconfig /all
```

Look for `DHCP Enabled . . . . . . . . . . : Yes` and a non-manual IP address.

## PowerShell Equivalent

```powershell
# Get the adapter index

$adapter = Get-NetAdapter -Name "Ethernet"

# Remove static IP and switch to DHCP
Set-NetIPInterface -InterfaceIndex $adapter.InterfaceIndex -Dhcp Enabled

# Remove any static IP entries
Remove-NetIPAddress -InterfaceIndex $adapter.InterfaceIndex -Confirm:$false

# Reset DNS to automatic
Set-DnsClientServerAddress -InterfaceIndex $adapter.InterfaceIndex -ResetServerAddresses

# Renew the lease
ipconfig /renew
```

## Checking DHCP Server Assignment

```cmd
:: Show DHCP lease details
ipconfig /all | findstr /i "DHCP\|IPv4\|Default Gateway\|DNS"
```

## Batch Script to Switch All Adapters to DHCP

```cmd
@echo off
:: Switch all Ethernet adapters to DHCP
for /f "tokens=*" %%A in ('netsh interface show interface ^| findstr /i "ethernet"') do (
    for /f "tokens=4" %%B in ("%%A") do (
        echo Switching %%B to DHCP...
        netsh interface ipv4 set address name="%%B" source=dhcp
        netsh interface ipv4 set dns name="%%B" source=dhcp
    )
)
ipconfig /release
ipconfig /renew
```

## Conclusion

`netsh interface ipv4 set address source=dhcp` switches the adapter to DHCP mode in one command. Always switch DNS to `source=dhcp` as well, then release/renew to obtain a fresh lease immediately.
