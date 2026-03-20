# How to Renew a DHCP Lease on Windows

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DHCP, Windows, Networking, Network Diagnostics, sysadmin

Description: Renewing a DHCP lease on Windows forces the system to release its current IP address and request a new one from the DHCP server, useful when changing networks or resolving IP conflict issues.

## Method 1: ipconfig (Command Prompt)

The most common method:

```cmd
REM Release the current DHCP lease
ipconfig /release

REM Wait a moment, then request a new lease
ipconfig /renew

REM For a specific adapter (replace "Wi-Fi" with your adapter name)
ipconfig /release "Wi-Fi"
ipconfig /renew "Wi-Fi"

REM View the new lease details
ipconfig /all
```

## Method 2: PowerShell

```powershell
# Release all DHCP leases
Get-NetAdapter | ForEach-Object {
    if ($_.Status -eq "Up") {
        ipconfig /release $_.Name
    }
}

# Renew all DHCP leases
Get-NetAdapter | ForEach-Object {
    if ($_.Status -eq "Up") {
        ipconfig /renew $_.Name
    }
}

# Alternative: release and renew specific adapter
$adapter = "Ethernet"
ipconfig /release $adapter
ipconfig /renew $adapter

# Verify new IP
Get-NetIPAddress -InterfaceAlias $adapter -AddressFamily IPv4
```

## Method 3: GUI

1. Press `Win + R`, type `ncpa.cpl`, press Enter.
2. Right-click the adapter → **Disable**.
3. Wait 5 seconds, then right-click → **Enable**.
4. The adapter will request a new lease on startup.

## Method 4: netsh

```cmd
REM Disable and re-enable DHCP on a specific adapter
netsh interface set interface "Ethernet" admin=disable
timeout /t 5
netsh interface set interface "Ethernet" admin=enable
```

## Flushing DNS Cache After Renewal

After getting a new IP, flush the DNS cache to avoid stale entries:

```cmd
ipconfig /flushdns
```

## Diagnosing Lease Issues

```cmd
REM Show full DHCP lease info
ipconfig /all | findstr /i "dhcp\|lease\|gateway\|dns"

REM Show DHCP event log in PowerShell
Get-WinEvent -LogName System | Where-Object { $_.Id -in @(1002, 1003, 50036) } | Select-Object -First 10
```

## Key Takeaways

- `ipconfig /release` followed by `ipconfig /renew` is the standard Windows lease renewal.
- Specify the adapter name if the machine has multiple network interfaces.
- Flush the DNS cache with `ipconfig /flushdns` after renewal to clear stale mappings.
- If renewal fails repeatedly, check DHCP server logs and verify network connectivity.
