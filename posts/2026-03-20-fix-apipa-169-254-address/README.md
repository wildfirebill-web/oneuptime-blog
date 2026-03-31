# How to Fix IPv4 Getting a 169.254.x.x (APIPA) Address

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: APIPA, 169.254, DHCP, Window, Troubleshooting

Description: Learn how to fix the issue where your network adapter gets a 169.254.x.x APIPA address instead of a proper DHCP-assigned IP, indicating DHCP failure.

## What Is APIPA?

APIPA (Automatic Private IP Addressing) is a fallback mechanism. When a device cannot contact a DHCP server after 60-90 seconds of trying, Windows/macOS/Linux automatically assigns itself an address in the 169.254.0.0/16 range.

A 169.254.x.x address means DHCP has failed completely.

## Step 1: Confirm APIPA Address

```cmd
ipconfig /all
REM Look for:
REM "Autoconfiguration IPv4 Address: 169.254.x.x"
REM This confirms DHCP failure
```

## Step 2: Check Physical Connectivity

```cmd
REM Verify link is up (cable connected / WiFi associated)
netsh interface show interface
REM Should show "Connected" status

REM For WiFi, verify association
netsh wlan show interfaces | findstr "State\|SSID"
```

## Step 3: Force DHCP Renewal

```cmd
REM Release APIPA address and request from DHCP
ipconfig /release
ipconfig /renew
ipconfig /all
```

## Step 4: Check DHCP Server

```bash
# On Linux/router: Check if DHCP server is running

systemctl status isc-dhcp-server
systemctl status dnsmasq

# Check if DHCP server is listening on the correct interface
ss -ulnp | grep 67

# Test DHCP from command line
sudo dhclient -v eth0
```

## Step 5: Check Firewall Blocking DHCP

```bash
# DHCP uses UDP ports 67 (server) and 68 (client)
# Verify firewall isn't blocking these

# Linux iptables
sudo iptables -L INPUT -n | grep -E "67|68"

# Allow DHCP
sudo iptables -I INPUT -p udp --dport 67 -j ACCEPT
sudo iptables -I INPUT -p udp --dport 68 -j ACCEPT
```

## Step 6: Disable APIPA (Windows) and Use Static IP

```cmd
REM Disable APIPA (Windows will fail silently instead of using 169.254.x.x)
REM Run Registry Editor
reg add "HKLM\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters\Interfaces\{GUID}" /v IPAutoconfigurationEnabled /t REG_DWORD /d 0 /f

REM Better approach: set static IP as a working alternative
netsh interface ip set address name="Ethernet" static 192.168.1.100 255.255.255.0 192.168.1.1
netsh interface ip set dns name="Ethernet" static 8.8.8.8
```

## Conclusion

169.254.x.x addresses mean DHCP is failing. Check physical connectivity first, then run `ipconfig /release && /renew`, verify the DHCP server is running with `systemctl status isc-dhcp-server`, and check for firewall rules blocking UDP 67/68. If DHCP is consistently unreliable, use a static IP as a workaround while investigating the DHCP server.
