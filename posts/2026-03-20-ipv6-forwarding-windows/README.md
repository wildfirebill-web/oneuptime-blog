# How to Enable IPv6 Forwarding on Windows

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Windows, IP Forwarding, Routing, netsh

Description: Learn how to enable IPv6 packet forwarding on Windows to allow a Windows Server to route IPv6 traffic between network interfaces.

## Overview

Windows Server can act as an IPv6 router by enabling IP forwarding. This allows the system to forward packets between network interfaces rather than dropping non-local packets. On Windows desktop editions, routing features are limited.

## Checking Current Forwarding State

```powershell
# Check if IPv6 forwarding is enabled
Get-NetIPInterface -AddressFamily IPv6 |
  Select-Object InterfaceAlias, Forwarding

# Output example:
# InterfaceAlias    Forwarding
# --------------    ----------
# Ethernet          Disabled
# Ethernet 2        Disabled
```

## Enabling IPv6 Forwarding with PowerShell

```powershell
# Enable forwarding on all IPv6 interfaces
Set-NetIPInterface -AddressFamily IPv6 -Forwarding Enabled

# Enable on a specific interface only
Set-NetIPInterface -InterfaceAlias "Ethernet" -AddressFamily IPv6 -Forwarding Enabled
Set-NetIPInterface -InterfaceAlias "Ethernet 2" -AddressFamily IPv6 -Forwarding Enabled

# Verify the change
Get-NetIPInterface -AddressFamily IPv6 |
  Select-Object InterfaceAlias, Forwarding
```

## Enabling Forwarding with netsh

```cmd
:: Enable IPv6 routing globally
netsh interface ipv6 set global routerdiscovery=enabled

:: Set forwarding on a specific interface
netsh interface ipv6 set interface "Ethernet" forwarding=enabled
netsh interface ipv6 set interface "Ethernet 2" forwarding=enabled

:: Verify
netsh interface ipv6 show interface "Ethernet"
```

## Using the Routing and Remote Access Service (RRAS)

For full router functionality on Windows Server, use RRAS:

```powershell
# Install the RRAS feature
Install-WindowsFeature -Name "Routing" -IncludeManagementTools

# Configure RRAS for LAN routing
$rrasCfg = [PSCustomObject]@{
    RouterType = "LAN"
}

# Enable RRAS with IPv6 routing
netsh routing ip install
```

Alternatively, use the GUI:
1. Open **Server Manager** → **Tools** → **Routing and Remote Access**
2. Right-click the server → **Configure and Enable Routing and Remote Access**
3. Select **Custom Configuration** → check **LAN routing**
4. Start the service

## Windows Registry Method (Advanced)

For non-RRAS environments, edit the registry:

```powershell
# Enable IP forwarding via registry (requires reboot)
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip6\Parameters" `
  -Name "IPEnableRouter" -Value 1 -Type DWord

# Restart the system or the TCP/IP stack to apply
Restart-Computer
```

## Verifying Packet Forwarding

```powershell
# Confirm routes are being forwarded
# From a client on one subnet, ping a host on another subnet
# The Windows router should forward the packets

# On the Windows router, check interface statistics
Get-NetAdapterStatistics | Select-Object Name, ReceivedPackets, SentPackets

# Use route print to verify the routing table has both subnets
route print -6
```

## Limitations on Windows Desktop

Windows 10/11 Home and Pro editions have IP forwarding disabled and it cannot be fully enabled without RRAS (which requires Server edition). For routing on desktop Windows, consider using a Linux VM or WSL2 with forwarding enabled.

## Summary

Enable IPv6 forwarding on Windows Server using `Set-NetIPInterface -AddressFamily IPv6 -Forwarding Enabled` or `netsh interface ipv6 set interface <name> forwarding=enabled`. For full routing capabilities, install the RRAS role. Verify forwarding is active with `Get-NetIPInterface` before testing inter-network packet flow.
