# How to Configure IPv6 on Windows DHCP Server

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Windows Server, DHCP, DHCPv6, PowerShell

Description: Learn how to configure DHCPv6 scopes on Windows Server DHCP to provide IPv6 addresses and network configuration to clients, including stateful and stateless DHCPv6.

## DHCPv6 Overview

DHCPv6 provides two modes:
- **Stateful**: Server assigns IPv6 addresses and options (like IPv4 DHCP)
- **Stateless**: SLAAC assigns addresses, DHCPv6 provides only options (DNS, etc.)

```text
Router RA → M flag = 1 → Clients use stateful DHCPv6 for addresses
Router RA → O flag = 1 → Clients use stateless DHCPv6 for options only
```

## Create a DHCPv6 Scope via PowerShell

```powershell
# Create a DHCPv6 scope (requires DHCP Server role installed)

Add-DhcpServerv6Scope `
    -Name "IPv6 Production Scope" `
    -Description "DHCPv6 scope for production network" `
    -Prefix "2001:db8::" `
    -State Active

# Verify scope was created
Get-DhcpServerv6Scope

# Set preferred and valid lifetimes
Set-DhcpServerv6Scope `
    -Prefix "2001:db8::" `
    -PreferredLifeTime (New-TimeSpan -Days 4) `
    -ValidLifeTime (New-TimeSpan -Days 8)
```

## Configure DHCPv6 Exclusions

```powershell
# Exclude a range from being assigned (for static addresses)
Add-DhcpServerv6ExclusionRange `
    -Prefix "2001:db8::" `
    -StartRange "2001:db8::1" `
    -EndRange "2001:db8::ff"

# View exclusions
Get-DhcpServerv6ExclusionRange -Prefix "2001:db8::"
```

## Configure DHCPv6 DNS Options

```powershell
# Set DNS server addresses for the scope
Set-DhcpServerv6OptionValue `
    -Prefix "2001:db8::" `
    -OptionId 23 `
    -Value "2001:4860:4860::8888", "2001:4860:4860::8844"

# Set DNS search domain (option 24)
Set-DhcpServerv6OptionValue `
    -Prefix "2001:db8::" `
    -OptionId 24 `
    -Value "example.com"

# View configured options
Get-DhcpServerv6OptionValue -Prefix "2001:db8::"
```

## Configure DHCPv6 via DHCP Manager GUI

```text
Steps:
1. Open DHCP Manager (dhcpmgmt.msc)

2. Expand server → IPv6

3. Right-click "IPv6" → New Scope...

4. In the wizard:
   - Scope Name: "Production IPv6"
   - Prefix: 2001:db8::
   - Preference: 0 (default)

5. Set exclusions for reserved addresses

6. Set lease duration (Preferred: 4 days, Valid: 8 days)

7. Configure options:
   - Option 00023 (DNS Recursive Name Server)
     Add: 2001:4860:4860::8888
```

## Create DHCPv6 Reservations

```powershell
# Reserve an address for a specific client (by DUID, not MAC)
# Get client DUID from lease table
Get-DhcpServerv6Lease -Prefix "2001:db8::"

# Add a reservation
Add-DhcpServerv6Reservation `
    -Prefix "2001:db8::" `
    -IPAddress "2001:db8::100" `
    -ClientDuid "00-01-00-01-12-34-56-78-00-11-22-33-44-55" `
    -Name "MyServer" `
    -Description "Reserved for MyServer"
```

## Monitor DHCPv6 Leases

```powershell
# View all active DHCPv6 leases
Get-DhcpServerv6Lease -Prefix "2001:db8::"

# Count active leases
(Get-DhcpServerv6Lease -Prefix "2001:db8::").Count

# View lease statistics
Get-DhcpServerv6ScopeStatistics -Prefix "2001:db8::"
```

## Summary

Configure DHCPv6 on Windows Server with `Add-DhcpServerv6Scope -Prefix "2001:db8::"` to create a scope. Add exclusions with `Add-DhcpServerv6ExclusionRange`. Set DNS options with `Set-DhcpServerv6OptionValue -OptionId 23`. Reservations use client DUIDs (not MAC addresses like DHCPv4). The router must send RAs with the M flag (managed) set to direct clients to use DHCPv6 for address assignment.
