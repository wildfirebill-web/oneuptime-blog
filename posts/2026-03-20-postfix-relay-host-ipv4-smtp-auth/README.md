# How to Set Up Postfix Relay Host Over IPv4 with SMTP Authentication

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Postfix, Relay Host, IPv4, SMTP Authentication, Email, Configuration

Description: Configure Postfix to relay all outbound mail through a specific IPv4 SMTP relay server with SASL authentication credentials.

## Introduction

Relaying through a smarthost or SaaS email provider (SendGrid, Amazon SES, Mailgun) is common for transactional email. Postfix forwards all outbound mail to the relay's IPv4 address with SMTP AUTH credentials.

## Basic Relay Host Configuration

```bash
# /etc/postfix/main.cf

# Relay all mail through this IPv4 address (brackets = skip MX lookup)

relayhost = [203.0.113.100]:587

# Or relay through a hostname
relayhost = [smtp.sendgrid.net]:587

# Enable SASL for authentication with the relay
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_sasl_security_options = noanonymous

# TLS for the relay connection
smtp_tls_security_level = encrypt
smtp_tls_CAfile = /etc/ssl/certs/ca-certificates.crt

# Force IPv4 for outbound connections
inet_protocols = ipv4
smtp_bind_address = 203.0.113.10   # Your server's outbound IP
```

## Setting Up SASL Credentials

```bash
# /etc/postfix/sasl_passwd
# Format: [host]:port  username:password
[203.0.113.100]:587    myuser:mypassword

# For named relay services:
[smtp.sendgrid.net]:587    apikey:SG.xxxxxxxxxxxx
[email-smtp.us-east-1.amazonaws.com]:587    AKIAIOSFODNN7EXAMPLE:secret
```

```bash
# Hash the credentials file
sudo postmap /etc/postfix/sasl_passwd

# Secure the credentials files
sudo chmod 600 /etc/postfix/sasl_passwd /etc/postfix/sasl_passwd.db
sudo chown root:root /etc/postfix/sasl_passwd /etc/postfix/sasl_passwd.db

# Reload Postfix
sudo postfix reload
```

## Relay with Specific Source IP for SPF

Ensure your SPF record at the relay matches your source:

```bash
# /etc/postfix/main.cf

# Relay host
relayhost = [smtp.relay.example.com]:587

# Use specific IP as source (must be in relay's SPF allowlist)
smtp_bind_address = 203.0.113.10

# TLS settings
smtp_tls_security_level = encrypt
smtp_tls_loglevel = 1   # Log TLS activity for debugging
```

## Testing the Relay Configuration

```bash
# Check configuration
sudo postconf relayhost
sudo postfix check

# Send test email
echo "Test relay" | mail -s "Relay Test" recipient@example.com

# Watch mail log for relay connection
sudo tail -f /var/log/mail.log

# Look for successful relay:
# postfix/smtp[...]: connect to smtp.relay.example.com[203.0.113.100]:587
# postfix/smtp[...]: AUTH: Login success
# postfix/smtp[...]: 250 2.0.0 OK: queued as ...

# If auth fails:
sudo tail -f /var/log/mail.log | grep "SASL\|auth\|relay"
```

## Relay Only Specific Domains

```bash
# /etc/postfix/main.cf

# Default: deliver direct
relayhost =

# Use transport map for specific domains
transport_maps = hash:/etc/postfix/transport
```

```bash
# /etc/postfix/transport
# Relay outbound mail for these domains through the smarthost
example.com   smtp:[203.0.113.100]:587
partner.com   smtp:[203.0.113.100]:587
```

```bash
sudo postmap /etc/postfix/transport
sudo postfix reload
```

## Conclusion

Postfix relay configuration requires four elements: `relayhost` with the IPv4 address in brackets (to skip MX lookup), `smtp_sasl_auth_enable = yes`, `smtp_sasl_password_maps` pointing to the credentials hash file, and `smtp_tls_security_level = encrypt` for secure authentication. Secure `sasl_passwd` with `chmod 600` since it contains plaintext credentials.
