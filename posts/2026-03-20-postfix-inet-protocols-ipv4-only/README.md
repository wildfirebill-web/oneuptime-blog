# How to Force Postfix to Use IPv4 Only with inet_protocols = ipv4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Postfix, IPv4, Inet_protocols, Email, Configuration, Dual Stack

Description: Configure Postfix to send and receive mail exclusively over IPv4 by setting inet_protocols = ipv4, preventing delivery failures on systems with broken IPv6 connectivity.

## Introduction

On dual-stack servers, Postfix may attempt SMTP delivery over IPv6 when AAAA records exist, causing timeouts or bounces if IPv6 routing is broken or remote servers don't support it. Setting `inet_protocols = ipv4` restricts all mail operations to IPv4.

## Setting inet_protocols

```bash
# /etc/postfix/main.cf

# Use only IPv4 for listening and delivery

inet_protocols = ipv4

# Apply changes
sudo postfix reload
```

## What inet_protocols Controls

| Setting | Behavior |
|---|---|
| `all` | Use both IPv4 and IPv6 |
| `ipv4` | Only IPv4 (A records) |
| `ipv6` | Only IPv6 (AAAA records) |
| `ipv4, ipv6` | Both (same as all) |

With `ipv4`:
- Postfix only listens on IPv4 sockets
- DNS lookups only use A records (not AAAA)
- All outbound SMTP uses IPv4
- All inbound SMTP on IPv4 only

## Verifying inet_protocols Is Applied

```bash
# Check current setting
postconf inet_protocols

# Verify listening sockets (IPv4 only)
sudo ss -tlnp | grep master
# Should show:
# LISTEN 0 100 0.0.0.0:25 0.0.0.0:*   ← IPv4 only
# Should NOT show:
# LISTEN 0 100 [::]:25  :::*           ← IPv6

# Check DNS lookup behavior in mail log
sudo tail -f /var/log/mail.log
# Should see "connect to smtp.gmail.com[142.250.x.x]:25" (IPv4)
# Should NOT see "[2607:f8b0:...]:25" (IPv6)
```

## Common Scenario: Delivery Failures Due to IPv6

```bash
# Signs of IPv6 delivery issues in /var/log/mail.log:
# connect to smtp.example.com[2001:db8::1]:25: Connection refused
# connect to smtp.example.com[203.0.113.10]:25: Connection established

# Fix: force IPv4 to skip IPv6 MX records
# /etc/postfix/main.cf
inet_protocols = ipv4
```

## Full IPv4-Only Configuration

```bash
# /etc/postfix/main.cf

# IPv4 only for all Postfix operations
inet_protocols = ipv4

# Bind inbound listener to specific IPv4
inet_interfaces = 203.0.113.10

# Bind outbound SMTP to specific IPv4
smtp_bind_address = 203.0.113.10

# Hostname resolving to correct IPv4
myhostname = mail.example.com

# SPF alignment: hostname A record must match smtp_bind_address
```

## Testing IPv4-Only Mail Delivery

```bash
# Send test mail and watch log
echo "IPv4 test" | mail -s "IPv4 Test" recipient@example.com
sudo tail -f /var/log/mail.log

# Check the "connect to" lines show IPv4 addresses only
grep "connect to" /var/log/mail.log | grep -v '\[2[0-9a-f]*:' | tail -5

# Verify with mail delivery test
swaks --to recipient@example.com --server 203.0.113.10 --port 25
```

## Conclusion

`inet_protocols = ipv4` is a single-line fix that prevents Postfix from using IPv6 for any mail operations. It is the recommended setting for servers where IPv6 is not fully configured or where IPv6 SMTP delivery to remote servers is unreliable. Always pair with `smtp_bind_address` and `inet_interfaces` for complete IPv4 control, and reload Postfix after changes.
