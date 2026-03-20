# How to Configure Postfix inet_protocols for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Postfix, IPv6, Email, Mail Server, SMTP, Linux

Description: Configure the Postfix inet_protocols parameter to enable IPv4-only, IPv6-only, or dual-stack email delivery and reception.

## Introduction

Postfix's `inet_protocols` setting controls which IP protocol version the mail server uses for both sending and receiving email. By default on many systems this is set to `ipv4` for safety, but modern mail infrastructure benefits from supporting IPv6.

## Understanding inet_protocols Options

The `inet_protocols` parameter in `/etc/postfix/main.cf` accepts three values:

| Value | Behavior |
|-------|----------|
| `ipv4` | Only use IPv4 for SMTP connections |
| `ipv6` | Only use IPv6 for SMTP connections |
| `all` | Use both IPv4 and IPv6 (dual-stack) |

## Viewing Current Configuration

Check the current setting before making changes:

```bash
# Show current inet_protocols value

postconf inet_protocols

# Show all Postfix configuration related to protocols
postconf | grep -i protocol
```

## Enabling Dual-Stack (Recommended)

To allow Postfix to send and receive email over both IPv4 and IPv6:

```bash
# Edit main.cf directly
sudo postconf -e 'inet_protocols = all'

# Reload Postfix to apply the change
sudo systemctl reload postfix

# Verify the setting took effect
postconf inet_protocols
```

## Enabling IPv6-Only

For an IPv6-only mail server (requires all DNS MX records to have AAAA records for recipients):

```bash
# Set to IPv6 only
sudo postconf -e 'inet_protocols = ipv6'
sudo systemctl reload postfix
```

## Restricting to IPv4-Only

If you experience delivery issues with IPv6 and want to disable it temporarily:

```bash
# Restrict to IPv4 only
sudo postconf -e 'inet_protocols = ipv4'
sudo systemctl reload postfix
```

## Verifying Postfix Listens on IPv6

After enabling IPv6, confirm Postfix is accepting connections on port 25 for IPv6:

```bash
# Check listening sockets for Postfix
ss -tlnp | grep ':25'

# With dual-stack or ipv6, you should see:
# LISTEN  0  100  :::25  :::*  users:(("master",pid=1234,fd=13))

# Test IPv6 SMTP connection locally
telnet -6 ::1 25
# Or using openssl for STARTTLS
openssl s_client -connect [::1]:25 -starttls smtp
```

## Checking Mail Logs for IPv6 Delivery

After enabling IPv6, verify outbound mail is using IPv6:

```bash
# Watch mail log for IPv6 connections
sudo tail -f /var/log/mail.log | grep -E "IPv6|2001:|::"

# Example log entry showing IPv6 delivery:
# postfix/smtp[1234]: connect to mail.example.com[2001:db8::25]:25
```

## Common Issues After Enabling inet_protocols = all

**DNS lookup failures**: Ensure your server has a functioning IPv6 DNS resolver. Test with:

```bash
dig AAAA google.com @::1
```

**No route to host for IPv6**: Your server may have an IPv6 address but no default route. Check:

```bash
ip -6 route show default
```

**Slow delivery**: If outbound IPv6 connectivity is broken, Postfix will fall back to IPv4 after a timeout. Consider setting `smtp_address_preference = ipv4` temporarily while debugging.

## Configuration Summary

A minimal dual-stack Postfix configuration in `/etc/postfix/main.cf`:

```ini
# Enable dual-stack IPv4 and IPv6
inet_protocols = all

# Optional: prefer IPv6 for outbound delivery
smtp_address_preference = ipv6
```

## Conclusion

The `inet_protocols = all` setting enables Postfix to participate fully in IPv6 email delivery. This is the recommended production setting for servers that have proper IPv6 connectivity, PTR records, and firewall rules in place.
