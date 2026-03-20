# How to Configure Exchange Server for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Exchange Server, IPv6, Email, Microsoft, SMTP, Windows Server

Description: Configure Microsoft Exchange Server to send and receive email over IPv6, including receive connectors, send connectors, and DNS configuration for IPv6 email delivery.

---

Exchange Server supports IPv6 for SMTP, as well as client access protocols. Proper IPv6 configuration enables Exchange to receive inbound email from IPv6 senders and deliver outbound mail to IPv6-capable recipients.

## Exchange Server IPv6 Overview

```
Exchange IPv6 requires:
- Windows Server with IPv6 enabled
- Exchange 2013/2016/2019 (full IPv6 support)
- DNS: AAAA record for mail server
- DNS: MX record pointing to mail server FQDN
- Reverse DNS (PTR) for IPv6 sending address
- IP allow lists and spam filter updates for IPv6
```

## Configuring Receive Connectors for IPv6

```powershell
# Open Exchange Management Shell

# Check existing receive connectors
Get-ReceiveConnector | Select Name, Bindings

# Set Internet receive connector to listen on IPv6
Set-ReceiveConnector "Default Frontend EXCHANGE-SERVER" `
  -Bindings "[::]:25","0.0.0.0:25"

# Or create new IPv6 receive connector
New-ReceiveConnector `
  -Name "IPv6 Internet Receive" `
  -Server EXCHANGE-SERVER `
  -Bindings "[::]:25" `
  -Usage Internet `
  -RemoteIPRanges "::-ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff"

# Verify connector binding
Get-ReceiveConnector "Default Frontend EXCHANGE-SERVER" | Select Bindings
```

## Configuring Send Connectors for IPv6

```powershell
# Enable IPv6 on outbound delivery
# Note: Exchange uses OS preferences for IPv4/IPv6

# Check current send connector
Get-SendConnector | Select Name, AddressSpaces, SourceTransportServers

# For IPv6 outbound: ensure OS prefers IPv6 for external delivery
# Check /etc/hosts equivalent in Windows for mail relay

# Prefer IPv6 for outbound (via OS registry on Windows)
# Or configure smart host using IPv6-capable host
Set-SendConnector "Internet Send Connector" `
  -SmartHosts "[2001:db8::relay.example.com]"
```

## DNS Configuration for IPv6 Email

```bash
# DNS records required for IPv6 email:

# A record (IPv4)
mail.example.com. IN A 203.0.113.1

# AAAA record (IPv6)
mail.example.com. IN AAAA 2001:db8::mail

# MX record (points to FQDN, not IP)
example.com. IN MX 10 mail.example.com.

# PTR record for IPv6 (reverse DNS - required for deliverability)
# Contact your ISP/upstream provider for IPv6 PTR delegation
1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.8.b.d.0.1.0.0.2.ip6.arpa. IN PTR mail.example.com.

# SPF record (include IPv6)
example.com. IN TXT "v=spf1 a mx ip4:203.0.113.1 ip6:2001:db8::mail/128 ~all"
```

## Exchange Client Access over IPv6

```powershell
# Configure OWA (Outlook Web App) for IPv6
# IIS binding - add IPv6 to Default Website
New-WebBinding `
  -Name "Default Web Site" `
  -Protocol "https" `
  -IPAddress "::" `
  -Port 443

# Exchange Web Services (EWS) over IPv6
# Follows IIS bindings automatically

# ActiveSync over IPv6
# Follows IIS bindings

# Verify IIS bindings
Get-WebBinding -Name "Default Web Site" |
  Select bindingInformation
```

## Spam and IP Reputation for IPv6

```powershell
# Add IPv6 IP Allow List to avoid false positives
Set-IPBlockListConfig -Enabled $true
Set-IPAllowListConfig -Enabled $true

# Add trusted IPv6 range to IP Allow List
Add-IPAllowListEntry `
  -IPRange "2001:db8:trusted::/48" `
  -Comment "Trusted IPv6 sending range"

# Check Content Filter for IPv6 issues
Get-ContentFilterConfig | Select BypassedRecipients, BypassedSenders

# Enable connection filtering for IPv6
Set-IPBlockListProvidersConfig -Enabled $true
```

## Testing Exchange IPv6

```bash
# Test SMTP over IPv6
telnet [2001:db8::mail-server] 25
# OR
nc -6 2001:db8::mail-server 25

# Send test email via IPv6 SMTP
swaks --to test@recipient.com \
  --from sender@example.com \
  --server [2001:db8::mail-server]:25 \
  --protocol ESMTP

# Verify email headers for IPv6
# Received: from [2001:db8::sender] by mail.example.com

# Check Exchange logs for IPv6 connections
Get-EventLog -LogName "Application" -Source "*MSExchange*" |
  Where-Object Message -match "2001:"
```

Exchange Server's IPv6 support through receive connector bindings and DNS AAAA records enables full participation in IPv6 email delivery, with the critical requirements being reverse DNS PTR records for the sending IPv6 address and SPF record updates to include the IPv6 sending address.
