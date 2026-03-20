# How to Troubleshoot Postfix Connection Timeouts on IPv4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Postfix, IPv4, Troubleshooting, Connection Timeout, SMTP, Email Delivery

Description: Diagnose and resolve Postfix SMTP connection timeouts on IPv4 caused by firewall blocks, DNS issues, IPv6 fallback failures, and incorrect timeout settings.

## Introduction

Postfix connection timeouts manifest as deferred mail with errors like "Connection timed out" or "No route to host." These are always network-level issues between your server and the destination's SMTP server.

## Reading Timeout Errors

```bash
# Check mail queue for timeout errors

sudo postqueue -p

# View detailed error
sudo postcat -qe <QUEUE_ID>

# Common timeout messages in /var/log/mail.log:
# connect to smtp.example.com[203.0.113.50]:25: Connection timed out
# SMTP error from remote mail server after initial connection:
#   timeout (message body too large, possible network issue)

sudo grep "timeout\|timed out" /var/log/mail.log | tail -20
```

## Step 1: Test Basic Connectivity

```bash
# Can you reach the remote SMTP server at all?
telnet smtp.gmail.com 25
# If "Connection timed out" → port 25 is blocked outbound

# Try alternate SMTP port (some ISPs block port 25)
telnet smtp.gmail.com 587
telnet smtp.gmail.com 465

# DNS resolution test
dig A smtp.gmail.com
# If no response → DNS issue
```

## Step 2: Check Outbound Port 25 Blocking

Many cloud providers and ISPs block outbound port 25 to prevent spam:

```bash
# Test if port 25 is reachable
nc -zv smtp.gmail.com 25 -w 5
# If timeout → port 25 outbound blocked

# AWS: port 25 blocked by default on EC2
# Fix: use a relay via SES or request port 25 unblocking

# Check iptables for outbound port 25 blocks
sudo iptables -L OUTPUT -n | grep 25
```

## Step 3: Verify IPv4 Routing

```bash
# Check routing to SMTP destination
traceroute smtp.gmail.com

# Verify source IP used for outbound connections
ip route get 8.8.8.8

# Test connectivity from Postfix's bound IP
curl --interface 203.0.113.10 http://www.google.com/
```

## Step 4: IPv6 Fallback Causing Delays

```bash
# Check if IPv6 is causing delays before IPv4 fallback
sudo tail -f /var/log/mail.log | grep "connect to"
# If you see IPv6 attempts before IPv4 → fix with:

# /etc/postfix/main.cf
inet_protocols = ipv4   # Skip IPv6 entirely

sudo postfix reload
sudo postqueue -f   # Retry deferred queue
```

## Step 5: Adjust Postfix Timeout Settings

```bash
# /etc/postfix/main.cf

# Reduce connection timeout (default: 30s is often fine)
smtp_connect_timeout = 30s

# Reduce delay between delivery attempts for deferred mail
maximal_queue_lifetime = 5d    # Total time to try before bouncing
minimal_backoff_time = 300s    # Min wait between retries (default 300s)
maximal_backoff_time = 4000s   # Max wait between retries

# For testing: reduce backoff to retry faster
minimal_backoff_time = 60s
```

## Step 6: Check Firewall on Both Ends

```bash
# Local firewall check
sudo iptables -L OUTPUT -n | grep -E "DROP|REJECT"
sudo iptables -L FORWARD -n | grep -E "DROP|REJECT"

# Test TCP connection with explicit source IP
sudo tcping -v 203.0.113.10 smtp.example.com 25
# Or
curl -v --interface 203.0.113.10 telnet://smtp.example.com:25
```

## Forcing Queue Flush After Fix

```bash
# After resolving the connectivity issue:
sudo postqueue -f    # Flush all deferred messages

# Watch for successful delivery
sudo tail -f /var/log/mail.log | grep "status=sent"
```

## Conclusion

Postfix IPv4 connection timeouts are always network problems: blocked port 25, routing failures, firewall rules, or DNS resolution issues. Start with `telnet smtp.destination.com 25` to isolate connectivity, check for IPv6 fallback failures with `inet_protocols = ipv4`, verify no outbound firewall blocks exist, and flush the deferred queue after resolving the issue.
