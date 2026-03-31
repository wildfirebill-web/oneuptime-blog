# How to Configure IPv6 DNS Servers on Windows

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Window, DNS, PowerShell, Network Configuration

Description: Learn how to configure IPv6 DNS server addresses on Windows using PowerShell, netsh, and the GUI, and verify that DNS resolution works over IPv6.

## Configure IPv6 DNS via PowerShell

```powershell
# Set IPv6 DNS servers (replaces existing DNS settings)

Set-DnsClientServerAddress -InterfaceAlias "Ethernet" `
    -ServerAddresses "2001:4860:4860::8888", "2001:4860:4860::8844", "8.8.8.8"

# Verify DNS settings
Get-DnsClientServerAddress -InterfaceAlias "Ethernet"

# Output:
# InterfaceAlias  Address Family  ServerAddresses
# --------------  --------------  ---------------
# Ethernet        IPv4            {8.8.8.8}
# Ethernet        IPv6            {2001:4860:4860::8888, 2001:4860:4860::8844}
```

## Configure IPv6 DNS via netsh

```cmd
:: Add IPv6 DNS servers
netsh interface ipv6 add dnsserver "Ethernet" 2001:4860:4860::8888 index=1
netsh interface ipv6 add dnsserver "Ethernet" 2001:4860:4860::8844 index=2

:: View current DNS servers
netsh interface ipv6 show dnsservers "Ethernet"

:: Delete a DNS server
netsh interface ipv6 delete dnsserver "Ethernet" 2001:4860:4860::8888

:: Set DNS to automatic (DHCP/RA)
netsh interface ipv6 set dnsservers "Ethernet" source=dhcp
```

## Configure IPv6 DNS via GUI

```sql
Steps:
1. Open ncpa.cpl (Win + R → ncpa.cpl)

2. Right-click adapter → Properties

3. Select "Internet Protocol Version 6 (TCP/IPv6)" → Properties

4. Select "Use the following DNS server addresses"

5. Enter:
   - Preferred DNS server: 2001:4860:4860::8888
   - Alternate DNS server: 2001:4860:4860::8844

6. Click OK
```

## Well-Known IPv6 DNS Servers

```text
Google Public DNS:
  Primary:   2001:4860:4860::8888
  Secondary: 2001:4860:4860::8844

Cloudflare DNS:
  Primary:   2606:4700:4700::1111
  Secondary: 2606:4700:4700::1001

Quad9 DNS:
  Primary:   2620:fe::fe
  Secondary: 2620:fe::9

OpenDNS:
  Primary:   2620:119:35::35
  Secondary: 2620:119:53::53
```

## Verify IPv6 DNS Resolution

```powershell
# Resolve a hostname using IPv6 DNS server
Resolve-DnsName -Name "google.com" -Type AAAA -Server "2001:4860:4860::8888"

# Test AAAA record resolution
Resolve-DnsName -Name "ipv6.google.com" -Type AAAA

# Check which DNS server is being used
Get-DnsClientServerAddress -AddressFamily IPv6

# Verify DNS transport over IPv6
nslookup google.com 2001:4860:4860::8888

# Check DNS connectivity
Test-NetConnection -ComputerName "2001:4860:4860::8888" -Port 53
```

## DNS Search Suffix Configuration

```powershell
# View current DNS suffix search list
Get-DnsClientGlobalSetting

# Set DNS suffix search list
Set-DnsClientGlobalSetting -SuffixSearchList "example.com", "corp.example.com"

# Set per-interface connection-specific DNS suffix
Set-DnsClient -InterfaceAlias "Ethernet" -ConnectionSpecificSuffix "example.com"
```

## Flushing DNS Cache After Changes

```powershell
# Flush DNS cache after changing servers
Clear-DnsClientCache

# Verify cache is cleared
Get-DnsClientCache

# ipconfig equivalent
ipconfig /flushdns
```

## Summary

Configure IPv6 DNS servers on Windows with `Set-DnsClientServerAddress -ServerAddresses "2001:4860:4860::8888"`, via `netsh interface ipv6 add dnsserver`, or through the **ncpa.cpl → IPv6 Properties** GUI. Use Google (`2001:4860:4860::8888`), Cloudflare (`2606:4700:4700::1111`), or Quad9 (`2620:fe::fe`) as IPv6 DNS providers. Verify with `Resolve-DnsName` and flush the cache with `Clear-DnsClientCache` after changes.
