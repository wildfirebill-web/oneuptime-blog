# How to Reset TCP/IP Stack on Windows Using netsh int ip reset

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Windows, Networking, netsh, TCP/IP, Reset, Troubleshooting

Description: Reset the Windows TCP/IP stack to its default state using netsh int ip reset to resolve persistent network connectivity issues caused by corrupted TCP/IP settings.

## Introduction

When Windows TCP/IP connectivity is broken in ways that IP reconfiguration cannot fix — such as after malware infection, bad driver installation, or registry corruption — resetting the TCP/IP stack rewrites all TCP/IP registry entries to their default state.

## When to Use This Command

Use `netsh int ip reset` when:
- All network connections fail despite correct IP/DNS/gateway settings
- `ping` to the loopback (`127.0.0.1`) fails
- TCP connections hang or fail with unusual error codes
- Network stack is suspected to be corrupted

## Running the Reset

Open an **elevated** (Administrator) command prompt:

```cmd
:: Reset TCP/IP stack and save the log
netsh int ip reset C:\tcpip-reset.log

:: Or without a log file
netsh int ip reset
```

Expected output:

```
Resetting Interface, OK!
Resetting Unicast Address, OK!
Resetting Neighbor, OK!
Resetting Path, OK!
Resetting , failed.
Access is denied.

Resetting Echo Request, OK!
...
Reset completed, reboot is required.
```

The "failed" line is normal — some entries require running as SYSTEM to reset. Reboot after running.

## Complete Network Stack Reset (Multiple Commands)

For a thorough reset, combine multiple `netsh` commands:

```cmd
:: Reset TCP/IP stack
netsh int ip reset

:: Reset Winsock catalog
netsh winsock reset

:: Clear ARP cache
netsh interface ip delete arpcache

:: Flush DNS cache
ipconfig /flushdns

:: Release and renew DHCP
ipconfig /release
ipconfig /renew

echo Reboot required for full effect.
```

## Save Reset Log for Analysis

```cmd
netsh int ip reset C:\tcpip-reset.log
type C:\tcpip-reset.log
```

The log shows every registry key that was reset.

## Rebooting After Reset

The reset takes effect only after a reboot:

```cmd
:: Reboot immediately
shutdown /r /t 5 /c "TCP/IP stack reset — rebooting"
```

## Checking Network Connectivity After Reboot

```cmd
:: Verify loopback first
ping 127.0.0.1

:: Ping gateway
ipconfig
ping 192.168.1.1

:: Test DNS
nslookup google.com
```

## PowerShell Alternative

```powershell
# Reset IP interfaces (less comprehensive than netsh, but available)
Get-NetIPAddress | Remove-NetIPAddress -Confirm:$false -ErrorAction SilentlyContinue
Get-NetRoute     | Remove-NetRoute     -Confirm:$false -ErrorAction SilentlyContinue

# Full stack reset still requires netsh
netsh int ip reset
netsh winsock reset
```

## Conclusion

`netsh int ip reset` is the nuclear option for Windows TCP/IP stack corruption — it rewrites all TCP/IP registry entries to defaults. Always combine it with `netsh winsock reset` and `ipconfig /flushdns`, then reboot. Verify connectivity systematically from loopback outward after the restart.
