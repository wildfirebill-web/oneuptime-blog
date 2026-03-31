# How to Configure Windows Firewall Rules for Specific IPv4 Addresses

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Window, Firewall, IPv4, Security, Netsh, PowerShell

Description: Create Windows Firewall rules that allow or block traffic to and from specific IPv4 addresses using netsh advfirewall and PowerShell New-NetFirewallRule.

## Introduction

Windows Defender Firewall rules can be scoped to specific IPv4 source or destination addresses. This lets you allow access only from trusted hosts, block specific IPs, and create precise security policies without affecting all traffic.

## Allow Inbound Traffic from a Specific IP

```cmd
:: Allow inbound TCP on port 22 only from 192.168.1.50
netsh advfirewall firewall add rule ^
    name="Allow SSH from 192.168.1.50" ^
    dir=in ^
    action=allow ^
    protocol=TCP ^
    localport=22 ^
    remoteip=192.168.1.50
```

## Allow Inbound from a Subnet

```cmd
:: Allow inbound on port 5432 from entire management subnet
netsh advfirewall firewall add rule ^
    name="PostgreSQL from mgmt subnet" ^
    dir=in ^
    action=allow ^
    protocol=TCP ^
    localport=5432 ^
    remoteip=10.0.0.0/8
```

## Block Outbound Traffic to a Specific IP

```cmd
:: Block all outbound traffic to a known malicious IP
netsh advfirewall firewall add rule ^
    name="Block 203.0.113.100" ^
    dir=out ^
    action=block ^
    remoteip=203.0.113.100
```

## PowerShell: New-NetFirewallRule

PowerShell provides richer firewall management:

```powershell
# Allow inbound HTTPS only from a specific IP range

New-NetFirewallRule `
    -Name "Allow HTTPS from 192.168.1.0/24" `
    -DisplayName "Allow HTTPS from Internal Subnet" `
    -Direction Inbound `
    -Protocol TCP `
    -LocalPort 443 `
    -RemoteAddress "192.168.1.0/24" `
    -Action Allow `
    -Profile Any `
    -Enabled True
```

## Listing Firewall Rules for a Specific IP

```cmd
:: Find rules involving a specific IP
netsh advfirewall firewall show rule name=all | findstr "192.168.1.50"
```

```powershell
# PowerShell - find rules with a specific remote address
Get-NetFirewallRule | Get-NetFirewallAddressFilter | Where-Object {$_.RemoteAddress -like "*192.168.1.50*"}
```

## Modifying an Existing Rule

```powershell
# Update the remote address on an existing rule
Set-NetFirewallRule `
    -Name "Allow SSH from 192.168.1.50" `
    -RemoteAddress "192.168.1.0/24"
```

## Deleting a Rule

```cmd
netsh advfirewall firewall delete rule name="Block 203.0.113.100"
```

```powershell
Remove-NetFirewallRule -Name "Block 203.0.113.100"
```

## Testing Firewall Rules

```powershell
# Test connectivity from the local machine to verify a rule is working
Test-NetConnection -ComputerName 192.168.1.100 -Port 443
```

From a remote host, verify access is allowed or denied as expected.

## Conclusion

Use `remoteip=` in `netsh advfirewall` or `-RemoteAddress` in `New-NetFirewallRule` to scope rules to specific IPv4 addresses or subnets. Always test from both allowed and blocked sources after creating rules to confirm the expected behavior.
