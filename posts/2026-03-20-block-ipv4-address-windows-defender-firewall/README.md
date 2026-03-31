# How to Block an IPv4 Address in Windows Defender Firewall

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Window, Firewall, IPv4, Security, Block, Netsh, PowerShell

Description: Block all inbound and outbound traffic to and from a specific IPv4 address in Windows Defender Firewall using netsh and PowerShell, and verify the block is effective.

## Introduction

Blocking a specific IPv4 address in Windows Defender Firewall is useful for isolating a compromised or suspicious host, blocking known malicious IPs, or enforcing network segmentation policies. Rules can block both inbound and outbound directions.

## Block All Inbound Traffic from a Specific IP

```cmd
:: Block all inbound connections from 203.0.113.100
netsh advfirewall firewall add rule ^
    name="Block Inbound 203.0.113.100" ^
    dir=in ^
    action=block ^
    remoteip=203.0.113.100 ^
    enable=yes
```

## Block All Outbound Traffic to a Specific IP

```cmd
:: Block all outbound connections to 203.0.113.100
netsh advfirewall firewall add rule ^
    name="Block Outbound 203.0.113.100" ^
    dir=out ^
    action=block ^
    remoteip=203.0.113.100 ^
    enable=yes
```

## Block Both Directions with One Script

```cmd
@echo off
set BLOCK_IP=203.0.113.100

netsh advfirewall firewall add rule name="Block IN %BLOCK_IP%"  dir=in  action=block remoteip=%BLOCK_IP%
netsh advfirewall firewall add rule name="Block OUT %BLOCK_IP%" dir=out action=block remoteip=%BLOCK_IP%

echo %BLOCK_IP% has been blocked in both directions.
```

## Block an Entire Subnet

```cmd
:: Block all traffic to/from a subnet
netsh advfirewall firewall add rule ^
    name="Block Subnet 10.99.0.0/16 IN" ^
    dir=in action=block remoteip=10.99.0.0/16

netsh advfirewall firewall add rule ^
    name="Block Subnet 10.99.0.0/16 OUT" ^
    dir=out action=block remoteip=10.99.0.0/16
```

## PowerShell Alternative

```powershell
# Block inbound

New-NetFirewallRule `
    -Name "Block 203.0.113.100 IN" `
    -DisplayName "Block Suspicious Host (Inbound)" `
    -Direction Inbound `
    -RemoteAddress "203.0.113.100" `
    -Action Block `
    -Enabled True

# Block outbound
New-NetFirewallRule `
    -Name "Block 203.0.113.100 OUT" `
    -DisplayName "Block Suspicious Host (Outbound)" `
    -Direction Outbound `
    -RemoteAddress "203.0.113.100" `
    -Action Block `
    -Enabled True
```

## Verifying the Block

```cmd
:: Confirm the rule exists
netsh advfirewall firewall show rule name="Block Inbound 203.0.113.100"

:: Test connectivity - should fail (timeout or unreachable)
ping 203.0.113.100
tracert 203.0.113.100
```

## Removing the Block

```cmd
netsh advfirewall firewall delete rule name="Block Inbound 203.0.113.100"
netsh advfirewall firewall delete rule name="Block Outbound 203.0.113.100"
```

```powershell
Remove-NetFirewallRule -Name "Block 203.0.113.100 IN"
Remove-NetFirewallRule -Name "Block 203.0.113.100 OUT"
```

## Conclusion

Create paired inbound and outbound block rules to fully isolate a specific IP address. Name rules descriptively so they are easy to audit and remove later. For bulk blocking (many IPs), consider using a Windows Firewall address set or a third-party firewall management tool.
