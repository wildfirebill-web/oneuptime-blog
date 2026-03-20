# How to Fix IPv4 and IPv6 Both Showing "Not Connected" on WiFi

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: WiFi, IPv4, IPv6, Not Connected, Windows, Troubleshooting

Description: Learn how to fix the issue where both IPv4 and IPv6 show "Not Connected" on Windows WiFi, indicating a complete connectivity failure at the IP layer.

## What Does "IPv4 and IPv6 Not Connected" Mean?

When Windows shows both IPv4 and IPv6 as "Not Connected" in the Network and Sharing Center, the WiFi adapter has associated with the access point (Layer 2 is up) but has failed to obtain any IP configuration. This is more severe than just "No Internet" — the device has no IP address at all.

## Step 1: Verify the WiFi Association

```cmd
REM Check if WiFi is connected at Layer 2
netsh wlan show interfaces

REM Output:
REM Name                   : Wi-Fi
REM State                  : connected     <- or disconnected
REM SSID                   : MyNetwork
REM BSSID                  : aa:bb:cc:dd:ee:ff
REM Authentication         : WPA2-Personal

REM If State is "connected" but no IP, the problem is at Layer 3
REM If State is "disconnected", fix the WiFi connection first
```

## Step 2: Attempt Manual DHCP Renewal

```cmd
REM Run as Administrator
ipconfig /release
ipconfig /renew

REM If renew fails with "Unable to contact DHCP server":
REM The DHCP server is not responding

REM Try release/renew for specific adapter
ipconfig /release "Wi-Fi"
ipconfig /renew "Wi-Fi"
```

## Step 3: Reset Network Components

```cmd
REM Full network stack reset (run as Administrator)
netsh winsock reset catalog
netsh int ip reset reset.log
netsh int ipv6 reset resetlog.log
netsh advfirewall reset
ipconfig /flushdns

REM Restart adapter
netsh interface set interface "Wi-Fi" disable
netsh interface set interface "Wi-Fi" enable

REM Reboot
shutdown /r /t 0
```

## Step 4: Re-enable IPv4 and IPv6 Protocol Bindings

Sometimes the protocol bindings get disabled:

```powershell
# PowerShell - Check bindings
Get-NetAdapterBinding -Name "Wi-Fi"

# Re-enable IPv4
Enable-NetAdapterBinding -Name "Wi-Fi" -ComponentID ms_tcpip

# Re-enable IPv6
Enable-NetAdapterBinding -Name "Wi-Fi" -ComponentID ms_tcpip6

# Via GUI:
# Control Panel → Network and Sharing Center
# Change adapter settings → Right-click Wi-Fi → Properties
# Ensure "Internet Protocol Version 4" and "Version 6" are checked
```

## Step 5: Check Windows Services

Required services must be running:

```cmd
REM Check DHCP Client service
sc query Dhcp
REM Should show: STATE : 4 RUNNING

REM Start if stopped
net start Dhcp

REM Check other required services
sc query NlaSvc    REM Network Location Awareness
sc query WlanSvc   REM WLAN AutoConfig
sc query Netprofm  REM Network List Service

REM Start any stopped services
net start NlaSvc
net start WlanSvc
```

## Step 6: Uninstall and Reinstall WiFi Adapter

```powershell
# Get the WiFi adapter
Get-NetAdapter -Name "Wi-Fi"

# Uninstall via Device Manager:
# devmgmt.msc → Network Adapters
# Right-click wireless adapter → Uninstall device
# Check "Delete the driver software for this device"
# Action → Scan for hardware changes (reinstalls)

# Or via PowerShell
$adapter = Get-PnpDevice -FriendlyName "*Wireless*" -Status "OK"
Disable-PnpDevice -InstanceId $adapter.InstanceId -Confirm:$false
Enable-PnpDevice -InstanceId $adapter.InstanceId -Confirm:$false
```

## Step 7: Check DHCP Server Availability

The issue may be the DHCP server, not the client:

```cmd
REM From another working device, check if DHCP server has capacity
REM Log into router and verify DHCP pool is not exhausted

REM Test if you can get an IP with a static assignment:
netsh interface ip set address name="Wi-Fi" static 192.168.1.100 255.255.255.0 192.168.1.1
netsh interface ip set dns name="Wi-Fi" static 8.8.8.8
ipconfig /all
REM If static works, DHCP server is the problem
```

## Conclusion

Both IPv4 and IPv6 showing "Not Connected" means the WiFi adapter is associated but has no IP address. Work through the steps: run `ipconfig /release && /renew`, then reset the TCP/IP stack with `netsh winsock reset && netsh int ip reset`, re-enable protocol bindings, verify the DHCP Client service is running, and reinstall the wireless adapter driver as a last resort. Test with a static IP to determine whether the issue is the DHCP client or DHCP server.
