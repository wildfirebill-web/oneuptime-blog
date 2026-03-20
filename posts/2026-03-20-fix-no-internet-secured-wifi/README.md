# How to Fix "No Internet, Secured" WiFi Error on Windows

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: WiFi, Windows, No Internet, Secured, Troubleshooting

Description: Learn how to fix the "No Internet, Secured" WiFi status in Windows, which means you're connected to WiFi but cannot access the internet.

## What "No Internet, Secured" Means

"Secured" means WPA2/WPA3 authentication succeeded — your password is correct.
"No Internet" means one of:
- DHCP failed (got 169.254.x.x APIPA address)
- Got a valid IP but DNS is broken
- Got a valid IP and DNS works but internet routing is blocked

## Step 1: Diagnose the Connection Layer

```cmd
REM Check what IP you actually have
ipconfig /all

REM 169.254.x.x = DHCP failed (APIPA)
REM 192.168.x.x or 10.x.x.x = DHCP worked
REM 0.0.0.0 = No IP at all

REM Test each layer
ping 127.0.0.1          REM TCP/IP stack
ping 192.168.1.1        REM Local gateway
ping 8.8.8.8            REM Internet (no DNS)
nslookup google.com     REM DNS test
```

## Step 2: Fix DHCP Failure (169.254.x.x)

```cmd
REM Release and renew DHCP
ipconfig /release
ipconfig /renew

REM If no IP obtained, reset network stack
netsh winsock reset
netsh int ip reset
shutdown /r /t 0
```

## Step 3: Fix DNS Issues

```powershell
# Change DNS to Google
Set-DnsClientServerAddress -InterfaceAlias "Wi-Fi" -ServerAddresses 8.8.8.8, 8.8.4.4
ipconfig /flushdns

# Test DNS
nslookup google.com 8.8.8.8
```

## Step 4: Disable and Re-enable Adapter

```powershell
# Restart WiFi adapter
Disable-NetAdapter -Name "Wi-Fi" -Confirm:$false
Start-Sleep 3
Enable-NetAdapter -Name "Wi-Fi"
```

## Step 5: Forget and Reconnect

1. Settings → Network & Internet → WiFi
2. Click the network → **Forget**
3. Reconnect and enter password

## Step 6: Reset NLA Service

```cmd
REM Restart Network Location Awareness service
net stop nlasvc
net start nlasvc
```

## Conclusion

"No Internet, Secured" is diagnosed by checking whether you have an IP (`ipconfig`), whether the gateway is reachable (`ping 192.168.1.1`), and whether DNS works (`nslookup`). Fix DHCP failures with `ipconfig /release && /renew`, DNS issues by setting `8.8.8.8`, and persistent issues with `netsh winsock reset` and reboot.
