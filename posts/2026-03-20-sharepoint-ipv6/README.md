# How to Configure SharePoint for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: SharePoint, IPv6, Microsoft, Enterprise, IIS, Windows Server

Description: Configure SharePoint Server to serve content to IPv6 clients, including IIS IPv6 bindings, alternate access mappings, and load balancer configuration.

---

SharePoint Server runs on IIS (Internet Information Services) on Windows Server. Enabling IPv6 access to SharePoint primarily involves configuring IIS bindings to listen on IPv6 addresses and updating Alternate Access Mappings (AAM) for proper URL resolution.

## IIS IPv6 Binding for SharePoint

```powershell
# Configure IIS to listen on IPv6 for SharePoint web application

# Open IIS Manager > Sites > SharePoint Web Application

# Method 1: IIS Manager GUI
# Sites > SharePoint - 80 > Bindings > Add
# Type: HTTP
# IP Address: All Unassigned (includes IPv6 via ::)
# OR specific: 2001:db8::sharepoint
# Port: 80
# Host Name: sharepoint.example.com

# Method 2: PowerShell
Import-Module WebAdministration

# Add IPv6 binding to SharePoint site
New-WebBinding `
  -Name "SharePoint - 80" `
  -Protocol "http" `
  -IPAddress "::" `
  -Port 80 `
  -HostHeader "sharepoint.example.com"

# Add HTTPS binding
New-WebBinding `
  -Name "SharePoint - 443" `
  -Protocol "https" `
  -IPAddress "::" `
  -Port 443 `
  -HostHeader "sharepoint.example.com"
```

## SharePoint Central Administration over IPv6

```powershell
# Check Central Admin current bindings
Get-WebBinding -Name "SharePoint Central Administration v4"

# Add IPv6 binding to Central Admin
New-WebBinding `
  -Name "SharePoint Central Administration v4" `
  -Protocol "http" `
  -IPAddress "::" `
  -Port 2016

# Verify bindings
Get-WebBinding -Name "SharePoint Central Administration v4" |
  Select-Object protocol, bindingInformation
```

## Alternate Access Mappings for IPv6

```text
Configure AAM in Central Administration:

1. Central Admin > Application Management
   > Configure Alternate Access Mappings

2. Add Internal URL:
   http://[2001:db8::sharepoint]

3. Add Public URL mapping:
   Zone: Default
   URL: https://sharepoint.example.com

Note: For SharePoint, using a FQDN with AAAA record
is preferred over raw IPv6 in AAM configuration
```

```powershell
# PowerShell for AAM configuration
Add-PSSnapin Microsoft.SharePoint.PowerShell -ErrorAction SilentlyContinue

# Add IPv6 alternate access mapping
$webapp = Get-SPWebApplication "http://sharepoint.example.com"
New-SPAlternateUrl `
  -WebApplication $webapp `
  -Url "http://[2001:db8::sharepoint]" `
  -Zone Default
```

## SharePoint and SQL Server over IPv6

```powershell
# SharePoint connects to SQL via connection string
# For IPv6 SQL, use SQL Server instance name that resolves to IPv6
# Or use a SQL Alias

# Create SQL Alias pointing to IPv6 SQL Server
# (64-bit SQL alias on 64-bit SharePoint)
# HKLM\SOFTWARE\Microsoft\MSSQLServer\Client\ConnectTo
# Value: tcp:[2001:db8::sql-server],1433

# Set-up alias via cliconfg.exe
# or via PowerShell registry manipulation
$regPath = "HKLM:\SOFTWARE\Microsoft\MSSQLServer\Client\ConnectTo"
Set-ItemProperty -Path $regPath `
  -Name "SHAREPOINTDB" `
  -Value "tcp:[2001:db8::sql-server],1433"
```

## Windows Firewall for SharePoint IPv6

```powershell
# Allow SharePoint HTTP/HTTPS over IPv6
New-NetFirewallRule `
  -DisplayName "SharePoint HTTP IPv6" `
  -Direction Inbound `
  -Protocol TCP `
  -LocalPort 80 `
  -Action Allow

New-NetFirewallRule `
  -DisplayName "SharePoint HTTPS IPv6" `
  -Direction Inbound `
  -Protocol TCP `
  -LocalPort 443 `
  -Action Allow

# Allow Central Administration
New-NetFirewallRule `
  -DisplayName "SharePoint CA IPv6" `
  -Direction Inbound `
  -Protocol TCP `
  -LocalPort 2016 `
  -Action Allow
```

## Verifying IPv6 Access to SharePoint

```powershell
# Test connectivity to SharePoint over IPv6
Test-NetConnection -ComputerName "2001:db8::sharepoint" -Port 80

# Verify IIS is listening on IPv6
netstat -an | Select-String ":80"
# Look for [::]:80

# Test HTTP access over IPv6
Invoke-WebRequest -Uri "http://[2001:db8::sharepoint]" -UseBasicParsing

# Check IIS access logs for IPv6 connections
Get-Content "C:\inetpub\logs\LogFiles\W3SVC1\*.log" |
  Select-String "2001:" | Select-Object -Last 10
```

SharePoint's IPv6 accessibility is achieved through IIS bindings on `::` (all IPv6 interfaces), with the recommendation to use FQDN-based access via AAAA DNS records rather than raw IPv6 addresses in AAM configuration for compatibility with SharePoint's internal URL normalization.
