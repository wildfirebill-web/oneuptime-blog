# How to Assign Static IPv4 Addresses on Windows

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Windows, IPv4, Networking, Network Configuration, Static IP, Sysadmin

Description: On Windows, static IPv4 addresses can be set through the GUI Network Adapter settings, via the netsh command-line tool, or using PowerShell's New-NetIPAddress cmdlet.

## Method 1: PowerShell (Recommended)

PowerShell provides the most scriptable and modern approach:

```powershell
# Find the interface index (InterfaceIndex column)

Get-NetAdapter

# Remove any existing IP configuration on the adapter (index 5 in this example)
Remove-NetIPAddress -InterfaceIndex 5 -Confirm:$false -ErrorAction SilentlyContinue
Remove-NetRoute -InterfaceIndex 5 -Confirm:$false -ErrorAction SilentlyContinue

# Assign a static IP address
New-NetIPAddress `
    -InterfaceIndex 5 `
    -IPAddress "192.168.1.100" `
    -PrefixLength 24 `
    -DefaultGateway "192.168.1.1"

# Set DNS servers
Set-DnsClientServerAddress `
    -InterfaceIndex 5 `
    -ServerAddresses ("8.8.8.8", "1.1.1.1")

# Verify
Get-NetIPAddress -InterfaceIndex 5
Get-NetRoute -InterfaceIndex 5
```

## Method 2: netsh (Command Prompt)

```cmd
REM Assign static IP (replace "Ethernet" with your adapter name)
netsh interface ipv4 set address name="Ethernet" static 192.168.1.100 255.255.255.0 192.168.1.1

REM Set DNS servers
netsh interface ipv4 set dns name="Ethernet" static 8.8.8.8
netsh interface ipv4 add dns name="Ethernet" 1.1.1.1 index=2

REM Verify
netsh interface ipv4 show config name="Ethernet"
```

## Method 3: GUI (Network Adapter Settings)

1. Open **Control Panel → Network and Sharing Center → Change adapter settings**.
2. Right-click the adapter → **Properties**.
3. Select **Internet Protocol Version 4 (TCP/IPv4)** → **Properties**.
4. Choose **Use the following IP address** and enter:
   - IP address: `192.168.1.100`
   - Subnet mask: `255.255.255.0`
   - Default gateway: `192.168.1.1`
5. Set preferred DNS server: `8.8.8.8`, alternate: `1.1.1.1`.
6. Click **OK** and close.

## Reverting to DHCP

```powershell
# PowerShell: switch back to DHCP
Set-NetIPInterface -InterfaceIndex 5 -Dhcp Enabled
Set-DnsClientServerAddress -InterfaceIndex 5 -ResetServerAddresses

# netsh equivalent
# netsh interface ipv4 set address name="Ethernet" dhcp
# netsh interface ipv4 set dns name="Ethernet" dhcp
```

## Verifying the Configuration

```powershell
# Show all IP configuration details
ipconfig /all

# Test gateway and internet connectivity
Test-NetConnection -ComputerName 192.168.1.1
Test-NetConnection -ComputerName 8.8.8.8 -Port 80
```

## Key Takeaways

- PowerShell (`New-NetIPAddress`) is the most scriptable and recommended method.
- `netsh` works on older Windows versions (XP through Windows 11).
- Always remove existing IP/route entries before assigning a new static address to avoid conflicts.
- Use `ipconfig /all` to verify the full IP, mask, gateway, and DNS configuration.
