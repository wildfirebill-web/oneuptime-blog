# How to Troubleshoot Postfix Not Sending Over IPv4 When IPv6 Fails

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Postfix, IPv4, IPv6, Troubleshooting, Email Delivery, SMTP

Description: Diagnose and fix Postfix mail delivery failures caused by failed IPv6 SMTP connections on dual-stack servers, forcing delivery over IPv4.

## Introduction

On dual-stack servers, Postfix may attempt IPv6 delivery first, fail (if IPv6 routing is broken or the remote server doesn't support it), and not fall back to IPv4—causing mail to queue indefinitely. This guide diagnoses and resolves these failures.

## Identifying IPv6 Delivery Failures

```bash
# Check the mail queue for deferred messages
sudo postqueue -p

# View detailed error for a specific message
sudo postcat -qe <QUEUE_ID>

# Check the mail log for IPv6 failure patterns
sudo grep "Connection refused\|Connection timed out\|Network unreachable" /var/log/mail.log | tail -20

# IPv6 failure indicators:
# connect to smtp.example.com[2001:db8::1]:25: Connection refused
# connect to smtp.example.com[2001:db8::1]:25: Network unreachable
# After this, Postfix should try IPv4 but sometimes doesn't
```

## Immediate Fix: Force IPv4

```bash
# /etc/postfix/main.cf
inet_protocols = ipv4

# Apply immediately
sudo postfix reload

# Flush deferred queue (retry all deferred messages)
sudo postqueue -f

# Watch log for successful delivery over IPv4
sudo tail -f /var/log/mail.log
# Should see: connect to smtp.example.com[203.0.113.x]:25
```

## Check if IPv6 Is Causing Issues

```bash
# Test IPv6 connectivity
ping6 smtp.gmail.com

# Test if IPv6 SMTP works
telnet -6 smtp.gmail.com 25
# If "Connection refused" or "Network unreachable" → IPv6 is broken

# Test IPv4 SMTP
telnet smtp.gmail.com 25
# Should connect successfully
```

## Fix DNS Resolution for IPv6-First Preference

Even with `inet_protocols = ipv4`, check if resolver prioritizes IPv6:

```bash
# Check which IP Postfix resolves for Gmail
postconf -e "smtp_address_preference = ipv4"

# Or check with dig
dig -4 A smtp.gmail.com      # Only A records (IPv4)
dig -6 AAAA smtp.gmail.com   # Only AAAA records (IPv6)
```

## Debugging Delivery

```bash
# Enable verbose SMTP delivery logs temporarily
postconf -e "debug_peer_list = smtp.gmail.com"
postconf -e "debug_peer_level = 3"
sudo postfix reload

# Send a test email and watch verbose log
echo "Debug test" | mail -s "Debug" test@gmail.com
sudo tail -f /var/log/mail.log

# Disable debug after investigation
postconf -e "debug_peer_list ="
postconf -e "debug_peer_level = 2"
sudo postfix reload
```

## Force IPv4 for Specific Destination Domains

Use transport maps for per-domain IPv4 forcing:

```bash
# /etc/postfix/transport
gmail.com     smtp4:
yahoo.com     smtp4:
hotmail.com   smtp4:
```

```bash
# /etc/postfix/master.cf
smtp4 unix - - n - - smtp
    -o inet_protocols=ipv4
    -o smtp_bind_address=203.0.113.10
```

```bash
sudo postmap /etc/postfix/transport
sudo postfix reload
```

## Conclusion

Postfix IPv6 delivery failures are solved by setting `inet_protocols = ipv4` in `main.cf` and reloading Postfix. This prevents any IPv6 SMTP attempts. After fixing, run `postqueue -f` to immediately retry deferred messages. For targeted fixes without disabling IPv6 globally, use transport maps with a custom `smtp4` service definition that overrides `inet_protocols` per destination domain.
