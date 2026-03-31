# How to Fix 'IPv4 Not Connected' Error on Windows

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv4, Window, Not Connected, Troubleshooting, Network

Description: Learn how to fix the 'IPv4 not connected' status in Windows Network and Sharing Center, covering TCP/IP stack resets, driver fixes, and DHCP troubleshooting.

## Step 1: Run the Network Troubleshooter

```cmd
REM Run Windows network troubleshooter
msdt.exe -id NetworkDiagnosticsNetworkAdapter
```

## Step 2: Reset TCP/IP Stack

```cmd
REM Run as Administrator
netsh winsock reset
netsh int ip reset
netsh int ipv6 reset
ipconfig /flushdns
ipconfig /release
ipconfig /renew
shutdown /r /t 0
```

## Step 3: Verify Protocol Bindings

```powershell
# Check if IPv4 is enabled on the adapter

Get-NetAdapterBinding -Name "Ethernet" | Where-Object {$_.ComponentID -eq "ms_tcpip"}

# Enable IPv4 if disabled
Enable-NetAdapterBinding -Name "Ethernet" -ComponentID ms_tcpip
```

## Step 4: Update or Roll Back Network Driver

1. Open Device Manager (`devmgmt.msc`)
2. Expand **Network Adapters**
3. Right-click adapter → **Update driver**
4. Or **Roll back driver** if issue started after a recent update

## Step 5: Set Static IP as Test

```powershell
# Test with a static IP to rule out DHCP issues
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 192.168.1.50 -PrefixLength 24 -DefaultGateway 192.168.1.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 8.8.8.8

# If this works, DHCP server has an issue
# Revert to DHCP when done
Set-NetIPInterface -InterfaceAlias "Ethernet" -Dhcp Enabled
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ResetServerAddresses
```

## Step 6: Re-register with DHCP Server

```cmd
REM Force DHCP re-registration
ipconfig /release
ping 127.0.0.1 -n 3 > nul    REM Brief pause
ipconfig /renew

REM Check if IP assigned
ipconfig /all
```

## Conclusion

"IPv4 not connected" is fixed by: first trying `ipconfig /release && /renew`, then `netsh winsock reset && netsh int ip reset` followed by reboot, enabling the IPv4 protocol binding via PowerShell, updating the NIC driver, and testing with a static IP to isolate DHCP server issues. Work through these steps in order until connectivity is restored.
