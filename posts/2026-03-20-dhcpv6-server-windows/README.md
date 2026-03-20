# How to Configure a DHCPv6 Server on Windows Server

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DHCPv6, IPv6, Windows Server, Networking, PowerShell, DHCP Role

Description: Learn how to install and configure the DHCPv6 server role on Windows Server to assign IPv6 addresses, configure options, and manage scopes using GUI and PowerShell.

---

Windows Server includes a built-in DHCP server role that supports both DHCPv4 and DHCPv6. This guide walks through installing the DHCP Server role, creating a DHCPv6 scope, configuring options, and managing the server with PowerShell.

---

## Installing the DHCP Server Role

### Using Server Manager (GUI)

1. Open **Server Manager** → **Add Roles and Features**
2. Select **Role-based or feature-based installation**
3. Choose **DHCP Server**
4. Complete the wizard and click **Install**
5. After installation, click **Complete DHCP configuration** to authorize the server in Active Directory

### Using PowerShell

```powershell
# Install DHCP Server role
Install-WindowsFeature -Name DHCP -IncludeManagementTools

# Authorize the DHCP server in Active Directory
Add-DhcpServerInDC -DnsName "dhcpserver.corp.example.com" -IPAddress 192.168.1.10

# Notify Server Manager that DHCP post-install config is done
Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\ServerManager\Roles\12" `
    -Name "ConfigurationState" -Value 2
```

---

## Creating a DHCPv6 Scope

### Using DHCP Console (GUI)

1. Open **DHCP Manager** (dhcpmgmt.msc)
2. Expand **IPv6** → right-click → **New Scope**
3. Enter scope name: `Production IPv6`
4. Set **Prefix**: `2001:db8:1::`
5. Set **Prefix Length**: `64`
6. Set **Preference**: `0` (default)
7. Configure Start/End address if needed (optional exclusions)
8. Set lease duration: T1 = 4 days, T2 = 6.4 days, Valid = 8 days
9. Activate the scope

### Using PowerShell

```powershell
# Create a DHCPv6 scope
Add-DhcpServerv6Scope -Name "Production IPv6" `
    -Prefix "2001:db8:1::" `
    -Preference 0 `
    -State Active

# Verify scope
Get-DhcpServerv6Scope
```

---

## Configuring DHCPv6 Options

```powershell
# Set DNS servers (option 23)
Set-DhcpServerv6OptionValue -ScopeId "2001:db8:1::" `
    -DnsServer "2001:db8:ff::53", "2001:4860:4860::8888"

# Set DNS domain search list (option 24)
Set-DhcpServerv6OptionValue -ScopeId "2001:db8:1::" `
    -DomainSearchList "corp.example.com", "internal.example.com"

# Verify options
Get-DhcpServerv6OptionValue -ScopeId "2001:db8:1::"
```

---

## Managing Address Exclusions

```powershell
# Exclude a range from the scope (for static assignments)
Add-DhcpServerv6ExclusionRange -Prefix "2001:db8:1::" `
    -StartRange "2001:db8:1::1" `
    -EndRange "2001:db8:1::99"

# View exclusions
Get-DhcpServerv6ExclusionRange -Prefix "2001:db8:1::"
```

---

## Creating Host Reservations

```powershell
# Reserve a specific address for a client by DUID
Add-DhcpServerv6Reservation -Prefix "2001:db8:1::" `
    -ClientDuid "0001000128abc123001122334455" `
    -Iaid 1 `
    -IPAddress "2001:db8:1::10" `
    -Name "webserver01"

# View reservations
Get-DhcpServerv6Reservation -Prefix "2001:db8:1::"
```

---

## Viewing Active Leases

```powershell
# Show all DHCPv6 leases
Get-DhcpServerv6Lease -Prefix "2001:db8:1::"

# Show leases with hostname
Get-DhcpServerv6Lease -Prefix "2001:db8:1::" | `
    Select-Object IPAddress, HostName, ClientDuid, LeaseExpiryTime

# Count active leases
(Get-DhcpServerv6Lease -Prefix "2001:db8:1::").Count
```

---

## DHCPv6 Server Statistics

```powershell
# Get server statistics
Get-DhcpServerv6Statistics

# Get scope statistics
Get-DhcpServerv6ScopeStatistics -Prefix "2001:db8:1::"
```

---

## Configuring DHCP Failover (Windows Server 2016+)

Windows Server DHCPv6 does not natively support DHCPv6 failover like DHCPv4. Use split scope as a workaround:

```powershell
# Server 1: lower half of addresses
Add-DhcpServerv6ExclusionRange -Prefix "2001:db8:1::" `
    -StartRange "2001:db8:1::201" -EndRange "2001:db8:1::3ff"

# Server 2: upper half
Add-DhcpServerv6ExclusionRange -Prefix "2001:db8:1::" `
    -StartRange "2001:db8:1::100" -EndRange "2001:db8:1::200"
```

---

## Troubleshooting

```powershell
# Check DHCP service status
Get-Service -Name DHCPServer

# Restart DHCP service
Restart-Service -Name DHCPServer

# Check Windows Event Log for DHCP events
Get-WinEvent -LogName "Microsoft-Windows-DHCP-Server/Operational" | `
    Select-Object -First 20 | Format-List

# Check if server is authorized
Get-DhcpServerInDC
```

---

## Best Practices

1. **Authorize the server in AD** before it serves leases
2. **Configure exclusions** for static IP ranges before activating scopes
3. **Set DNS options** — clients need name servers to function
4. **Monitor scope utilization** with `Get-DhcpServerv6ScopeStatistics`
5. **Back up DHCP database** regularly with `Backup-DhcpServer`

---

## Conclusion

Windows Server's DHCP role provides a fully functional DHCPv6 server with GUI and PowerShell management. Create scopes, configure options, set reservations, and monitor leases — all from the DHCP console or a PowerShell session.

---

*Monitor your Windows Server infrastructure with [OneUptime](https://oneuptime.com) — full-stack monitoring with IPv6 support.*
