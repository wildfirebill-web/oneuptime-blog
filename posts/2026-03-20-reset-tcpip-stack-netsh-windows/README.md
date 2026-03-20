# How to Reset TCP/IP Stack with netsh on Windows

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Netsh, TCP/IP, Windows, Network Reset, Winsock

Description: Learn how to use netsh commands to fully reset the Windows TCP/IP stack and Winsock catalog, fixing corrupted network configurations that cause persistent connectivity failures.

## When to Reset the TCP/IP Stack

Reset the TCP/IP stack when you experience:
- No internet despite being connected
- "Ethernet doesn't have a valid IP configuration"
- Connections that appear active but don't transfer data
- Issues after malware removal or failed VPN uninstall
- Network problems that persist after driver updates

## Step 1: Full TCP/IP and Winsock Reset

```cmd
REM Run Command Prompt as Administrator (crucial - will fail otherwise)

REM Reset Winsock catalog (fixes LSP/layered service provider corruption)
netsh winsock reset catalog

REM Reset IPv4 TCP/IP stack
netsh int ip reset reset.log

REM Reset IPv6 TCP/IP stack
netsh int ipv6 reset reset6.log

REM Flush DNS resolver cache
ipconfig /flushdns

REM Re-register DNS names
ipconfig /registerdns

REM Release and renew DHCP lease
ipconfig /release
ipconfig /renew

REM REBOOT IS REQUIRED
shutdown /r /t 0
```

## Step 2: Verify Reset Log

```cmd
REM After reboot, review what was reset
type reset.log
type reset6.log

REM Example output:
REM Reseting Interface, OK!
REM Reseting Unicast Address, OK!
REM Reseting Routes, OK!
```

## Step 3: Using PowerShell Alternative

```powershell
# PowerShell equivalent (Windows 8/10/11)

# Run as Administrator

# Reset network stack
netsh winsock reset
netsh int ip reset

# Remove all IP addresses and routes for a clean slate
Get-NetAdapter | ForEach-Object {
    $name = $_.Name
    Remove-NetIPAddress -InterfaceAlias $name -Confirm:$false -ErrorAction SilentlyContinue
    Remove-NetRoute -InterfaceAlias $name -DestinationPrefix "0.0.0.0/0" -Confirm:$false -ErrorAction SilentlyContinue
}

# Flush DNS
Clear-DnsClientCache

# Verify DNS cache is empty
Get-DnsClientCache
```

## Step 4: Reset Network Using Windows Settings (GUI)

```text
Settings → System → Troubleshoot → Other troubleshooters
→ Internet Connections → Run

Or for full reset:
Settings → Network & Internet → Advanced network settings
→ Network reset → Reset now

WARNING: Network reset removes all Wi-Fi passwords and VPN configurations
```

## Step 5: Reset Individual Components

```cmd
REM Reset only Winsock (less disruptive)
netsh winsock reset

REM Reset only IP routing table
netsh int ip reset

REM Reset only firewall rules to defaults
netsh advfirewall reset

REM Disable IPv6 if causing conflicts (rare)
netsh int ipv6 set global disabled
```

## Step 6: Check What Gets Reset

The `netsh int ip reset` command modifies these registry keys:

```cmd
REM Keys affected by netsh int ip reset:
REM HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters
REM HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\DHCP\Parameters

REM View current values before reset (for documentation)
reg query "HKLM\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters"
```

## Step 7: Post-Reset Verification

```cmd
REM Verify TCP/IP stack is functional after reboot
ipconfig /all

REM Test connectivity layers
ping 127.0.0.1          REM TCP/IP stack
ping 192.168.1.1        REM Gateway
ping 8.8.8.8            REM Internet
nslookup google.com     REM DNS

REM Check Winsock providers are clean
netsh winsock show catalog
REM Should show only Microsoft providers, no third-party LSPs from old VPN/AV
```

## Conclusion

The full TCP/IP reset sequence is `netsh winsock reset catalog` + `netsh int ip reset reset.log` + `ipconfig /flushdns` + reboot. This fixes most persistent Windows network issues by returning the TCP/IP stack to a clean state. Check the `reset.log` file after reboot to confirm which settings were changed. The network stack reset via Settings → Network reset is the most thorough option but removes all saved Wi-Fi credentials.
