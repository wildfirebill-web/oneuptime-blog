# How to Fix "DHCP Is Not Enabled for WiFi" Error

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DHCP, WiFi, Windows, Troubleshooting, Network

Description: Learn how to fix the "DHCP is not enabled for WiFi" error message that appears when running the Windows network troubleshooter, by re-enabling automatic IP addressing.

## What Causes This Error?

This error appears when the WiFi adapter is configured to use a static (manual) IP address instead of DHCP. The Windows troubleshooter detects this and reports it as the likely cause of connectivity issues.

## Step 1: Re-enable DHCP via Settings

1. Settings → Network & Internet → WiFi
2. Click the connected network → Properties
3. Under IP settings, click **Edit**
4. Change from **Manual** to **Automatic (DHCP)**
5. Click Save

## Step 2: Re-enable DHCP via PowerShell

```powershell
# Run as Administrator

# Re-enable DHCP for Wi-Fi adapter
Set-NetIPInterface -InterfaceAlias "Wi-Fi" -Dhcp Enabled

# Clear any static DNS settings
Set-DnsClientServerAddress -InterfaceAlias "Wi-Fi" -ResetServerAddresses

# Release and renew
ipconfig /release
ipconfig /renew

# Verify DHCP is now used
Get-NetIPInterface -InterfaceAlias "Wi-Fi" | Select-Object Dhcp
# Should show: Enabled
```

## Step 3: Use netsh Command

```cmd
REM Re-enable DHCP for Wi-Fi
netsh interface ip set address name="Wi-Fi" dhcp
netsh interface ip set dns name="Wi-Fi" dhcp

REM Verify
netsh interface ip show config name="Wi-Fi"
REM Should show: "DHCP Enabled: Yes"
```

## Step 4: Check Group Policy (Corporate Networks)

On domain-joined machines, group policy may force static IPs:

```cmd
REM Check applied group policies
gpresult /r | findstr /i "ip\|network"
```

Contact your IT administrator if group policy is forcing a static IP.

## Step 5: Reset Adapter and Renew

```powershell
# Disable adapter
Disable-NetAdapter -Name "Wi-Fi" -Confirm:$false
Start-Sleep -Seconds 3

# Re-enable (ensures fresh start)
Enable-NetAdapter -Name "Wi-Fi"
Start-Sleep -Seconds 5

# Renew DHCP
ipconfig /renew "Wi-Fi"
ipconfig /all
```

## Conclusion

"DHCP is not enabled" means the WiFi adapter has a static IP configured. Fix via Settings → Network → WiFi → Properties → IP settings → Automatic (DHCP), or via PowerShell with `Set-NetIPInterface -Dhcp Enabled`. On corporate/domain machines, verify group policy isn't enforcing static IPs. After enabling DHCP, run `ipconfig /renew` to obtain a new address from the DHCP server.
