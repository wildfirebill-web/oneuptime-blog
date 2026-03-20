# How to Configure Postfix smtp_bind_address for IPv4 Outbound Mail

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Postfix, IPv4, Smtp_bind_address, Outbound Mail, SMTP, Email

Description: Configure Postfix smtp_bind_address to control which IPv4 address is used for outbound SMTP connections, ensuring mail comes from the correct IP for SPF compliance.

## Introduction

When a server has multiple IPv4 addresses, Postfix may send mail from an unexpected IP. `smtp_bind_address` forces all outbound SMTP connections to originate from a specific IPv4, which is critical for SPF record alignment.

## Setting smtp_bind_address

```bash
# /etc/postfix/main.cf

# Force all outbound SMTP connections to use this IPv4

smtp_bind_address = 203.0.113.10

# For IPv6, use smtp_bind_address6 (leave empty to disable IPv6 sending)
smtp_bind_address6 =

# Force IPv4 only (important on dual-stack servers)
inet_protocols = ipv4
```

Apply the changes:

```bash
sudo postfix reload
```

## Verifying the Source IP

```bash
# Send a test email and check the source IP in mail headers
echo "Test" | mail -s "Test" test@example.com

# Check outbound SMTP connection source
sudo tcpdump -i eth0 port 25 -n | head -10
# Source IP should be 203.0.113.10

# Check mail log
sudo tail -f /var/log/mail.log | grep smtp
```

## SPF Record Alignment

Your SPF record must include the `smtp_bind_address` IP:

```dns
; DNS TXT record for your domain
example.com. IN TXT "v=spf1 ip4:203.0.113.10 ~all"

; Or include the server's hostname (which resolves to the IP)
example.com. IN TXT "v=spf1 a:mail.example.com ~all"
```

## Per-Destination Binding with transport_maps

Use different source IPs for different mail destinations:

```bash
# /etc/postfix/main.cf
smtp_bind_address = 203.0.113.10   # Default outbound IP

# Per-destination IP via transport maps
transport_maps = hash:/etc/postfix/transport
```

```bash
# /etc/postfix/transport
# Domain        transport
gmail.com       smtp_via_11:
yahoo.com       smtp_via_12:
```

```bash
# /etc/postfix/master.cf
smtp_via_11 unix - - n - - smtp
    -o smtp_bind_address=203.0.113.11

smtp_via_12 unix - - n - - smtp
    -o smtp_bind_address=203.0.113.12
```

```bash
postmap /etc/postfix/transport
sudo postfix reload
```

## Testing SMTP Binding

```bash
# Test SMTP connection with telnet
telnet 10.0.0.1 25
EHLO mail.example.com
MAIL FROM:<test@example.com>

# Check which IP appears in mail headers when received
# Received: from mail.example.com (mail.example.com [203.0.113.10])

# Verify SMTP binding in Postfix logs
sudo postqueue -f
sudo tail -f /var/log/mail.log
```

## Conclusion

`smtp_bind_address` in Postfix controls which IPv4 the SMTP client (delivery agent) uses for outbound connections. Set it to the IP listed in your SPF record to ensure alignment. On dual-stack servers, also set `inet_protocols = ipv4` to prevent unexpected IPv6 delivery. Use `master.cf` service definitions with `-o smtp_bind_address=...` for per-destination IP selection.
