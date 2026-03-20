# How to Set Up Postfix Rate Limiting Per IPv4 Client Address

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Postfix, Rate Limiting, IPv4, Anti-Spam, SMTP, Security

Description: Implement per-IPv4 SMTP rate limiting in Postfix using anvil service, smtpd restrictions, and policyd to prevent abuse and reduce spam delivery attempts.

## Introduction

Rate limiting prevents a single IPv4 address from flooding your mail server with connections or messages. Postfix provides built-in rate limiting via the `anvil` service and per-client connection limits.

## Built-in Anvil Rate Limiting

```bash
# /etc/postfix/main.cf

# Connection rate limits (anvil service)
smtpd_client_connection_rate_limit = 30    # Max 30 connections per minute per IP
smtpd_client_connection_count_limit = 10   # Max 10 concurrent connections per IP
smtpd_client_message_rate_limit = 30       # Max 30 messages per minute per IP
smtpd_client_recipient_rate_limit = 100    # Max 100 recipients per minute per IP

# Exempt trusted networks from rate limiting
smtpd_client_event_limit_exceptions = $mynetworks
```

## Connection Count Limiting

```bash
# /etc/postfix/main.cf

# Limit concurrent SMTP connections per IP
smtpd_client_connection_count_limit = 5

# Apply to specific restrictions
smtpd_client_restrictions =
    permit_mynetworks
    check_client_access cidr:/etc/postfix/client_rate
    permit
```

## Using Policyd-SPF or Postscreen for Rate Limiting

Postscreen provides advanced rate limiting at the connection level:

```bash
# /etc/postfix/main.cf

# Enable postscreen (runs before smtpd)
postscreen_access_list = permit_mynetworks cidr:/etc/postfix/postscreen_access.cidr
postscreen_blacklist_action = drop
postscreen_greet_action = enforce
postscreen_dnsbl_action = enforce
postscreen_dnsbl_sites = zen.spamhaus.org*3

# Rate limits via postscreen
postscreen_non_smtp_command_action = drop
postscreen_pipelining_action = enforce
```

## External Policy Daemon with policyd

For advanced per-IP rate limiting:

```bash
# Install policyd (or postfwd)
sudo apt install postfix-policyd-spf-python  # Example

# /etc/postfix/main.cf
smtpd_recipient_restrictions =
    permit_mynetworks
    permit_sasl_authenticated
    reject_unauth_destination
    check_policy_service inet:127.0.0.1:10031  # policyd

# Start policyd service
sudo systemctl start policyd
```

## Monitoring Rate Limit Events

```bash
# Watch for rate limit rejections in mail log
sudo tail -f /var/log/mail.log | grep -E "rate limit|too many|NOQUEUE"

# Common rate limit messages:
# NOQUEUE: reject: RCPT from ...: 450 4.7.1 <client>: Client host rejected: too many connections
# NOQUEUE: reject: RCPT from ...: 450 4.7.1 ...: too many messages

# Count rejections per IP
grep "too many" /var/log/mail.log | \
  grep -oE '\[([0-9.]+)\]' | sort | uniq -c | sort -rn | head 10

# View anvil statistics
sudo tail -f /var/log/mail.log | grep anvil
```

## Whitelisting High-Volume Trusted Senders

```bash
# /etc/postfix/client_rate (CIDR format)
# Trusted IPs: unlimited connections
10.0.0.0/8        OK
192.168.0.0/16    OK
# Default: rate limiting applies
```

```bash
# /etc/postfix/main.cf
smtpd_client_event_limit_exceptions = $mynetworks
# mynetworks hosts are exempt from all connection rate limits
```

## Conclusion

Postfix anvil rate limiting (`smtpd_client_connection_rate_limit`, `smtpd_client_message_rate_limit`, etc.) provides built-in per-IPv4 throttling without external dependencies. Set reasonable limits based on your expected legitimate traffic volume, use `smtpd_client_event_limit_exceptions` to whitelist internal networks, and monitor mail logs for rate limit events to tune thresholds. For advanced rate limiting, use postscreen or an external policy daemon.
