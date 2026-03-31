# How to Configure IPv6 MTU on Windows Interfaces

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, MTU, Window, Network Configuration, Netsh

Description: Configure and verify IPv6 MTU settings on Windows network interfaces using netsh and PowerShell, understand MTU inheritance, and troubleshoot MTU-related issues.

## Introduction

Windows manages IPv6 MTU settings through the `netsh` command and PowerShell's `NetIPInterface` cmdlets. Unlike Linux where MTU is a single per-interface value, Windows tracks MTU separately for IPv4 and IPv6. Tunnel adapters (Teredo, ISATAP, 6to4) have their MTU auto-calculated, but physical interfaces may need manual adjustment when used with VPNs or tunnels.

## Viewing Current MTU Settings

```powershell
# PowerShell: View MTU for all IPv6 interfaces

Get-NetIPInterface -AddressFamily IPv6 | Select-Object InterfaceAlias, NlMtu, ConnectionState

# View a specific interface
Get-NetIPInterface -InterfaceAlias "Ethernet" -AddressFamily IPv6

# View all adapter properties including MTU
Get-NetAdapter | Select-Object Name, InterfaceDescription, LinkLayerAddress, MtuSize

# Check MTU from the command line using netsh
netsh interface ipv6 show subinterfaces

# Detailed interface information
netsh interface ipv6 show interface "Ethernet"

# Show all IPv6 interfaces and their MTU
netsh interface ipv6 show interfaces
```

## Setting IPv6 MTU on Windows

```powershell
# PowerShell: Set IPv6 MTU on an interface
Set-NetIPInterface -InterfaceAlias "Ethernet" -AddressFamily IPv6 -NlMtu 1480

# Using netsh (works on older Windows versions too)
netsh interface ipv6 set subinterface "Ethernet" mtu=1480

# For a tunnel adapter
netsh interface ipv6 set subinterface "Teredo Tunneling Pseudo-Interface" mtu=1280

# Verify the change
Get-NetIPInterface -InterfaceAlias "Ethernet" -AddressFamily IPv6 | Select-Object NlMtu

# List interfaces by index (useful when alias names have spaces)
Get-NetIPInterface -AddressFamily IPv6 | Select-Object InterfaceIndex, InterfaceAlias, NlMtu

# Set by interface index
Set-NetIPInterface -InterfaceIndex 12 -AddressFamily IPv6 -NlMtu 1480
```

## Checking MTU for PMTU Discovery

```powershell
# Check the PMTU cache (destination cache) in Windows
netsh interface ipv6 show destinationcache

# Show entries for a specific destination
netsh interface ipv6 show destinationcache | Where-Object { $_ -match "2001:db8" }

# Flush the destination (PMTU) cache
netsh interface ipv6 delete destinationcache

# Test connectivity with ping (Windows ping6 is just 'ping -6')
# Test with large packet size (1472 = 1500 - 20 IPv4 - 8 ICMP)
ping -6 -l 1452 -f 2001:db8::1
# -l = packet data size
# -f = don't fragment (equivalent to ping6 -M do on Linux)
```

## MTU for VPN and Tunnel Interfaces

```powershell
# WireGuard MTU configuration (set in WireGuard config file)
# Usually auto-detected; can be set in [Interface] section:
# MTU = 1420

# Check WireGuard interface MTU after connection
Get-NetIPInterface -InterfaceAlias "WireGuard Tunnel" -AddressFamily IPv6

# OpenVPN: Set MTU in the .ovpn config file
# tun-mtu 1500
# mssfix 1420

# For Windows built-in VPN (PPTP/L2TP/IKEv2):
# MTU is typically set automatically; override if needed:
$vpnAdapter = Get-NetAdapter | Where-Object { $_.InterfaceDescription -like "*WAN Miniport*" }
Set-NetIPInterface -InterfaceIndex $vpnAdapter.ifIndex -AddressFamily IPv6 -NlMtu 1280
```

## Scripted MTU Audit

```powershell
# Audit all IPv6 interfaces for proper MTU
$interfaces = Get-NetIPInterface -AddressFamily IPv6

foreach ($iface in $interfaces) {
    $status = if ($iface.NlMtu -ge 1280) { "OK" } else { "BROKEN (< 1280)" }
    $connected = if ($iface.ConnectionState -eq "Connected") { "Connected" } else { "Disconnected" }

    Write-Output ("{0,-40} MTU={1,-6} [{2}] {3}" -f `
        $iface.InterfaceAlias, `
        $iface.NlMtu, `
        $status, `
        $connected)
}
```

## Conclusion

Windows IPv6 MTU configuration uses either `Set-NetIPInterface` in PowerShell or `netsh interface ipv6 set subinterface` in the command prompt. The minimum valid MTU for IPv6 is 1280 bytes - interfaces below this cannot carry IPv6 traffic. When using VPNs or tunnels, calculate the overhead and reduce the interface MTU accordingly. The destination cache (`netsh interface ipv6 show destinationcache`) shows cached PMTU values for specific destinations.
