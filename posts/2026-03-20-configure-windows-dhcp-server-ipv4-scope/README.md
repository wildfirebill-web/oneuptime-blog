# How to Configure a Windows DHCP Server for IPv4 Scope

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Windows, DHCP, IPv4, Windows Server, Network Configuration, PowerShell

Description: Configure an IPv4 DHCP scope on Windows Server using the DHCP Manager GUI and PowerShell, including address range, subnet mask, gateway, DNS options, and lease duration.

## Introduction

A DHCP scope defines the pool of IPv4 addresses a Windows DHCP server can assign to clients, along with options like the default gateway and DNS servers. Configuring scopes correctly is foundational to dynamic network management.

## Prerequisites

- Windows Server with DHCP Server role installed
- The DHCP server authorized in Active Directory (if domain-joined)

## Installing the DHCP Role (PowerShell)

```powershell
# Install DHCP Server role

Install-WindowsFeature DHCP -IncludeManagementTools

# Authorize the DHCP server in Active Directory
Add-DhcpServerInDC -DnsName "dhcpserver.corp.example.com" -IPAddress 192.168.1.10
```

## Creating a Scope via PowerShell

```powershell
# Add a new IPv4 scope
Add-DhcpServerv4Scope `
    -Name "Main Office Subnet" `
    -StartRange 192.168.1.100 `
    -EndRange 192.168.1.200 `
    -SubnetMask 255.255.255.0 `
    -State Active `
    -LeaseDuration (New-TimeSpan -Days 1)
```

## Setting Scope Options (Gateway and DNS)

```powershell
# Set default gateway (option 3)
Set-DhcpServerv4OptionValue `
    -ScopeId 192.168.1.0 `
    -OptionId 3 `
    -Value 192.168.1.1

# Set DNS servers (option 6)
Set-DhcpServerv4OptionValue `
    -ScopeId 192.168.1.0 `
    -OptionId 6 `
    -Value @("8.8.8.8", "1.1.1.1")

# Set DNS domain name (option 15)
Set-DhcpServerv4OptionValue `
    -ScopeId 192.168.1.0 `
    -OptionId 15 `
    -Value "corp.example.com"
```

## Adding Exclusion Ranges

Exclude static IP ranges from the pool (for servers, printers, network devices):

```powershell
# Exclude 192.168.1.1 through 192.168.1.99 from the scope
Add-DhcpServerv4ExclusionRange `
    -ScopeId 192.168.1.0 `
    -StartRange 192.168.1.1 `
    -EndRange 192.168.1.99
```

## Verifying the Scope

```powershell
# Show all scopes
Get-DhcpServerv4Scope

# Show scope options
Get-DhcpServerv4OptionValue -ScopeId 192.168.1.0

# Show current leases
Get-DhcpServerv4Lease -ScopeId 192.168.1.0
```

## GUI: DHCP Manager

1. Open **Server Manager** → **Tools** → **DHCP**
2. Expand the server node
3. Right-click **IPv4** → **New Scope**
4. Follow the wizard:
   - Set the scope name
   - Enter start/end IP and mask
   - Add exclusion ranges
   - Set lease duration
   - Set gateway (option 3) and DNS (option 6)
5. Activate the scope

## Conclusion

PowerShell's `Add-DhcpServerv4Scope` and `Set-DhcpServerv4OptionValue` provide scriptable DHCP scope configuration. Always add exclusion ranges for statically-assigned devices, set gateway and DNS options, and verify with `Get-DhcpServerv4Lease` to confirm clients are receiving addresses.
