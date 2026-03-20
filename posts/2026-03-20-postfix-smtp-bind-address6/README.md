# How to Configure Postfix smtp_bind_address6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Postfix, IPv6, Email, SMTP, Mail Server, Configuration

Description: Configure Postfix smtp_bind_address6 to control which IPv6 source address is used when delivering outbound email to remote mail servers.

## Introduction

When a server has multiple IPv6 addresses, Postfix may choose any of them as the source address for outbound SMTP connections. The `smtp_bind_address6` parameter pins outbound SMTP to a specific IPv6 address - important for reputation management and ensuring correct PTR/rDNS alignment.

## Why smtp_bind_address6 Matters

Mail servers check your sending IP against:
- PTR (reverse DNS) records
- SPF `ip6:` mechanisms
- IP reputation databases

If your server has multiple IPv6 addresses and sends from a random one, PTR alignment will fail for some messages, hurting deliverability.

## Finding Your Available IPv6 Addresses

Before configuring, list available IPv6 addresses on the server:

```bash
# List all IPv6 addresses

ip -6 addr show

# Focus on global scope addresses only (exclude link-local fe80::)
ip -6 addr show scope global

# Example output:
# inet6 2001:db8::10/64 scope global
# inet6 2001:db8::11/64 scope global
```

## Setting smtp_bind_address6

Edit `/etc/postfix/main.cf` to pin outbound delivery to a specific IPv6 address:

```bash
# Set the IPv6 source address for outbound SMTP
sudo postconf -e 'smtp_bind_address6 = 2001:db8::10'

# If you also want to pin IPv4, set both
sudo postconf -e 'smtp_bind_address = 203.0.113.10'
sudo postconf -e 'smtp_bind_address6 = 2001:db8::10'

# Reload Postfix
sudo systemctl reload postfix
```

## Verifying the Binding

Send a test email and check the mail log to confirm the correct source IP is used:

```bash
# Send a test message
echo "Test body" | mail -s "IPv6 bind test" test@example.com

# Watch the mail log for the connection
sudo tail -f /var/log/mail.log | grep smtp

# Expected output showing source address:
# postfix/smtp[1234]: connect from local[2001:db8::10] to remote[2001:db8::25]:25
```

## Multi-Instance Setup with Different Bindings

If you run multiple Postfix instances (e.g., for transactional vs. bulk email), each can bind to a different IPv6 address:

```bash
# Instance for transactional email
# /etc/postfix-transactional/main.cf
smtp_bind_address6 = 2001:db8::10

# Instance for bulk/marketing email
# /etc/postfix-bulk/main.cf
smtp_bind_address6 = 2001:db8::11
```

## Configuring PTR Records for the Bound Address

For best deliverability, ensure the bound IPv6 address has a PTR record matching your mail hostname:

```bash
# Verify PTR record for the bound address
# Replace with your actual IPv6 in reverse notation
dig -x 2001:db8::10 +short
# Expected: mail.example.com.

# Check from the mail server itself
postconf myhostname
# mail.example.com
```

The PTR record for `2001:db8::10` should return `mail.example.com`, and `mail.example.com` should have an AAAA record pointing to `2001:db8::10`.

## Resetting to Default

To revert to letting the OS choose the source address:

```bash
# Empty string means use OS default routing
sudo postconf -e 'smtp_bind_address6 ='
sudo systemctl reload postfix
```

## Common Issues

**"Cannot assign requested address"**: The IPv6 address you specified in `smtp_bind_address6` is not configured on any local interface. Double-check with `ip -6 addr show`.

**Delivery via IPv4 even with bind set**: Ensure `smtp_address_preference` is set to `ipv6` or `any`, and that `inet_protocols = all` or `ipv6`.

## Conclusion

`smtp_bind_address6` is a critical Postfix parameter for IPv6 mail servers with multiple addresses. By pinning outbound delivery to a specific IPv6 address with matching PTR and SPF records, you ensure consistent mail reputation and deliverability.
