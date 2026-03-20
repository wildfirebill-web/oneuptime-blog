# How to Configure IPv6 in Hyper-V

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Hyper-V, Windows Server, Virtualization, Virtual Networking

Description: Configure IPv6 for Hyper-V virtual switches, VM network adapters, and host management over IPv6, with PowerShell configuration and verification commands.

## Introduction

Hyper-V on Windows Server supports IPv6 for both host management traffic and virtual machine networking. Hyper-V Virtual Switches bridge physical network adapters to VMs, and IPv6 flows through virtual switches transparently when the physical network supports it. VMs receive IPv6 addresses through SLAAC, DHCPv6, or static configuration in the guest OS.

## Create Hyper-V Virtual Switch with IPv6

```powershell
# Create an external virtual switch (bridges to physical NIC)

# IPv6 passes through transparently
New-VMSwitch -Name "ExternalSwitch" `
    -NetAdapterName "Ethernet" `
    -AllowManagementOS $true

# Create an internal virtual switch for host-VM communication over IPv6
New-VMSwitch -Name "InternalSwitch" -SwitchType Internal

# Configure IPv6 on the host adapter for the internal switch
$adapter = Get-NetAdapter | Where-Object {$_.Name -like "*InternalSwitch*"}
New-NetIPAddress -InterfaceIndex $adapter.InterfaceIndex `
    -IPAddress "fd00::1" `
    -PrefixLength 64
```

## Configure IPv6 on Hyper-V Host Management

```powershell
# List all network adapters and their IPv6 addresses
Get-NetIPAddress -AddressFamily IPv6 | Where-Object {$_.PrefixOrigin -ne "WellKnown"}

# Add static IPv6 to the host management adapter
New-NetIPAddress -InterfaceAlias "Ethernet" `
    -IPAddress "2001:db8::10" `
    -PrefixLength 64 `
    -DefaultGateway "2001:db8::1"

# Add IPv6 DNS server
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" `
    -ServerAddresses "2001:db8::53", "2001:4860:4860::8888"

# Verify IPv6 configuration
Get-NetIPAddress -InterfaceAlias "Ethernet" -AddressFamily IPv6
```

## Assign Static IPv6 to a VM

```powershell
# Inside the VM (PowerShell in guest OS)

# List current IP configuration
Get-NetIPAddress -AddressFamily IPv6

# Assign static IPv6 address
$interface = Get-NetAdapter | Where-Object {$_.Status -eq "Up"}
New-NetIPAddress -InterfaceIndex $interface.InterfaceIndex `
    -IPAddress "2001:db8::100" `
    -PrefixLength 64 `
    -DefaultGateway "2001:db8::1"

# Add IPv6 DNS
Set-DnsClientServerAddress -InterfaceIndex $interface.InterfaceIndex `
    -ServerAddresses "2001:db8::53"

# Verify
Test-NetConnection -ComputerName "2001:db8::1" -TraceRoute
```

## Hyper-V Live Migration over IPv6

```powershell
# Configure Live Migration to use IPv6 network
# On the Hyper-V host:

# List Live Migration networks
Get-VMHost | Select-Object -ExpandProperty VirtualMachineMigrationEnabled

# Enable Live Migration
Enable-VMMigration

# Set Live Migration to use specific IPv6 network
Add-VMMigrationNetwork -Cidr "2001:db8:vmotion::/48"

# List configured migration networks
Get-VMMigrationNetwork

# Move a VM using IPv6 Live Migration
Move-VM -Name "MyVM" -DestinationHost "2001:db8::20" -IncludeStorage
```

## VM Network Adapter IPv6 Configuration (PowerShell)

```powershell
# Hyper-V VM network adapter settings

# Get VM network adapters
Get-VMNetworkAdapter -VMName "MyVM"

# Enable MAC address spoofing (sometimes needed for nested IPv6 NDP)
Set-VMNetworkAdapter -VMName "MyVM" -MacAddressSpoofing On

# Check VM IPv6 addresses reported by Hyper-V (requires guest integration services)
Get-VM -Name "MyVM" | Get-VMNetworkAdapter | Select-Object -ExpandProperty IPAddresses

# Should show both IPv4 and IPv6 addresses if guest integration services are installed
```

## Windows Server DHCPv6 for Hyper-V VMs

```powershell
# Install DHCP server role
Install-WindowsFeature DHCP -IncludeManagementTools

# Create DHCPv6 scope for VMs
Add-DhcpServerv6Scope `
    -Name "VM IPv6 Scope" `
    -Prefix "2001:db8:vms::" `
    -State Active

# Add DNS option to DHCPv6 scope
Set-DhcpServerv6OptionValue `
    -ScopeId "2001:db8:vms::" `
    -OptionId 23 `
    -Value "2001:db8::53"

# Verify DHCPv6 scope
Get-DhcpServerv6Scope
Get-DhcpServerv6ScopeStatistics
```

## Firewall for Hyper-V Management over IPv6

```powershell
# Windows Firewall: allow management access over IPv6

# Allow WinRM over IPv6
New-NetFirewallRule -DisplayName "Allow WinRM IPv6" `
    -Direction Inbound `
    -Protocol TCP `
    -LocalPort 5985,5986 `
    -AddressFamily IPv6 `
    -Action Allow

# Allow Live Migration over IPv6
New-NetFirewallRule -DisplayName "Allow Live Migration IPv6" `
    -Direction Inbound `
    -Protocol TCP `
    -LocalPort 6600 `
    -AddressFamily IPv6 `
    -Action Allow
```

## Verify IPv6 Connectivity

```powershell
# Test IPv6 reachability to Hyper-V host
Test-NetConnection -ComputerName "2001:db8::10" -Port 443

# Ping VM over IPv6
ping -6 2001:db8::100

# Check Hyper-V host networking
Get-VMNetworkAdapter -All | Select-Object Name, SwitchName, IPAddresses

# Check VM reports IPv6 via Hyper-V integration services
Get-VM | Get-VMNetworkAdapter | Where-Object {$_.IPAddresses -match ":"}
```

## Conclusion

Hyper-V supports IPv6 for host management networks (configured via PowerShell `New-NetIPAddress`), VM network adapters (which pass IPv6 through virtual switches transparently), and Live Migration (configured with `Add-VMMigrationNetwork` using IPv6 CIDRs). Virtual machines receive IPv6 via SLAAC from the connected network or DHCPv6 from a Windows Server DHCP scope. Hyper-V Integration Services report VM IPv6 addresses to the host, enabling `Get-VMNetworkAdapter` to display current VM IPv6 assignments. Windows Firewall rules should be added with `-AddressFamily IPv6` to permit management and migration traffic over IPv6.
