# How to Configure Postfix smtp_address_preference for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Postfix, IPv6, SMTP, Email Delivery, Mail Configuration, Linux

Description: Configure Postfix smtp_address_preference to control whether outbound SMTP connections prefer IPv4 or IPv6 when both are available for a destination.

## Introduction

When `inet_protocols = all` is set in Postfix, both IPv4 and IPv6 may be available for connecting to a remote mail server. The `smtp_address_preference` parameter controls which protocol is tried first, affecting delivery performance, reputation, and fallback behavior.

## Understanding smtp_address_preference Values

| Value | Behavior |
|-------|----------|
| `any` | Try IPv6 first, then IPv4 (default on many systems) |
| `ipv4` | Always prefer IPv4 over IPv6 |
| `ipv6` | Always prefer IPv6 over IPv4 |

## Viewing the Current Setting

```bash
# Check current preference setting
postconf smtp_address_preference

# Check along with related settings
postconf smtp_address_preference inet_protocols smtp_bind_address6
```

## Setting IPv6 Preference

To prioritize IPv6 for outbound delivery when available:

```bash
# Prefer IPv6 for outbound SMTP
sudo postconf -e 'smtp_address_preference = ipv6'
sudo systemctl reload postfix
```

This is the recommended setting for production servers with proper IPv6 configuration (PTR records, SPF ip6: mechanisms, etc.).

## Setting IPv4 Preference (Fallback/Transition)

During IPv6 migration or when IPv6 deliverability is uncertain:

```bash
# Prefer IPv4 for outbound SMTP (safer during transition)
sudo postconf -e 'smtp_address_preference = ipv4'
sudo systemctl reload postfix
```

## Using "any" for Balanced Delivery

The `any` setting lets Postfix use whatever the OS resolver returns first, which typically follows Happy Eyeballs-like behavior:

```bash
sudo postconf -e 'smtp_address_preference = any'
sudo systemctl reload postfix
```

## Verifying Delivery Protocol in Logs

After changing the preference, send a test message and check the delivery protocol:

```bash
# Send test email
echo "Protocol preference test" | sendmail -v test@gmail.com

# Watch the mail log
sudo tail -n 50 /var/log/mail.log | grep -E "connect to|IPv6|relay"

# Example showing IPv6 delivery:
# postfix/smtp[1234]: Trusted TLS connection established to
#   gmail-smtp-in.l.google.com[2607:f8b0:4003:c08::1a]:25
```

## Full Dual-Stack Configuration Example

A complete dual-stack Postfix configuration in `/etc/postfix/main.cf`:

```ini
# Protocol settings
inet_protocols = all

# Prefer IPv6 for outbound connections
smtp_address_preference = ipv6

# Bind outbound SMTP to specific IPv6 address
smtp_bind_address6 = 2001:db8::10

# Bind outbound SMTP to specific IPv4 address
smtp_bind_address = 203.0.113.10
```

## Handling Destination-Specific Preferences

For granular control, use Postfix transport maps to route specific domains over IPv4 or IPv6:

```bash
# /etc/postfix/transport
# Send to example.com via IPv4 relay (useful for ISPs that block IPv6)
example.com  smtp:[203.0.113.1]:25

# Apply the transport map
sudo postconf -e 'transport_maps = hash:/etc/postfix/transport'
sudo postmap /etc/postfix/transport
sudo systemctl reload postfix
```

## Monitoring Protocol Usage

Use the mail log to build a picture of which protocol is being used most:

```bash
# Count IPv6 vs IPv4 deliveries in the last hour
sudo grep "$(date '+%b %e %H')" /var/log/mail.log | \
    grep "connect to" | \
    awk '{print $NF}' | \
    grep -oE '\[.*\]' | \
    while read addr; do
        if echo "$addr" | grep -q ':'; then echo "IPv6"; else echo "IPv4"; fi
    done | sort | uniq -c
```

## Conclusion

`smtp_address_preference` is a simple but impactful Postfix parameter. Setting it to `ipv6` on a fully configured dual-stack mail server takes advantage of modern IPv6 delivery paths, while `ipv4` provides a safety net during initial IPv6 deployment.
