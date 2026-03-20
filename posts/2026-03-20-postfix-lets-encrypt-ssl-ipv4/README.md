# How to Set Up Postfix with Let's Encrypt SSL on an IPv4 Server

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Postfix, Let's Encrypt, SSL, TLS, IPv4, Email, Certbot, Security

Description: Learn how to configure Postfix to use a free Let's Encrypt TLS certificate on an IPv4 mail server for encrypted SMTP connections.

---

Using a Let's Encrypt certificate with Postfix provides trusted TLS encryption for your mail server without the "certificate not trusted" warnings associated with self-signed certs.

## Prerequisites

- A domain (`mail.example.com`) pointing to your server's IPv4 address.
- Port 80 open for the ACME HTTP-01 challenge.

## Obtaining the Certificate

```bash
# Install Certbot
apt install certbot -y  # Debian/Ubuntu

# Obtain a certificate for the mail hostname
# Use standalone mode (Certbot runs its own web server on port 80)
certbot certonly --standalone -d mail.example.com

# Or if you already have a web server running on port 80, use webroot mode:
# certbot certonly --webroot -w /var/www/html -d mail.example.com
```

Certificates are stored in `/etc/letsencrypt/live/mail.example.com/`.

## Configuring Postfix to Use the Certificate

```ini
# /etc/postfix/main.cf

# --- IPv4 only ---
inet_protocols = ipv4

# --- TLS: use Let's Encrypt certificate ---
# For incoming connections (clients connecting to your server)
smtpd_tls_cert_file = /etc/letsencrypt/live/mail.example.com/fullchain.pem
smtpd_tls_key_file  = /etc/letsencrypt/live/mail.example.com/privkey.pem
smtpd_tls_CAfile    = /etc/letsencrypt/live/mail.example.com/chain.pem
smtpd_use_tls = yes
smtpd_tls_security_level = may       # Offer TLS; don't require it for port 25
smtpd_tls_protocols = !SSLv2,!SSLv3,!TLSv1,!TLSv1.1
smtpd_tls_loglevel = 1

# --- TLS: for outgoing connections (Postfix → other mail servers) ---
smtp_tls_cert_file = /etc/letsencrypt/live/mail.example.com/fullchain.pem
smtp_tls_key_file  = /etc/letsencrypt/live/mail.example.com/privkey.pem
smtp_tls_CAfile    = /etc/ssl/certs/ca-certificates.crt
smtp_tls_security_level = may        # Use TLS when available
smtp_tls_protocols = !SSLv2,!SSLv3,!TLSv1,!TLSv1.1
smtp_tls_loglevel = 1

# TLS session cache (improves performance)
smtpd_tls_session_cache_database = btree:${data_directory}/smtpd_scache
smtp_tls_session_cache_database  = btree:${data_directory}/smtp_scache
```

## Granting Postfix Access to Let's Encrypt Files

Let's Encrypt private keys are in `/etc/letsencrypt/live/`, which is only readable by root. Add Postfix to the group or adjust permissions.

```bash
# Add the postfix user to the ssl-cert group (if it exists)
usermod -aG ssl-cert postfix

# Or use a deploy hook to copy certs to a Postfix-readable location
cat > /etc/letsencrypt/renewal-hooks/deploy/postfix.sh << 'EOF'
#!/bin/bash
cp /etc/letsencrypt/live/mail.example.com/fullchain.pem /etc/postfix/ssl/
cp /etc/letsencrypt/live/mail.example.com/privkey.pem   /etc/postfix/ssl/
chown root:postfix /etc/postfix/ssl/*.pem
chmod 640 /etc/postfix/ssl/*.pem
systemctl reload postfix
EOF
chmod +x /etc/letsencrypt/renewal-hooks/deploy/postfix.sh
```

## Testing TLS

```bash
# Check Postfix config
postfix check

# Reload Postfix
systemctl reload postfix

# Test TLS handshake on port 25
openssl s_client -starttls smtp -connect mail.example.com:25

# Verify certificate details
openssl s_client -starttls smtp -connect mail.example.com:25 2>&1 | grep -E "subject|issuer|CN"
```

## Automatic Certificate Renewal

```bash
# Test renewal
certbot renew --dry-run

# The deploy hook will automatically reload Postfix after renewal
```

## Key Takeaways

- Use `smtpd_tls_cert_file` and `smtp_tls_cert_file` to configure TLS for both incoming and outgoing connections.
- Use a deploy hook to copy certificates to a Postfix-readable location after renewal.
- `smtpd_tls_security_level = may` offers TLS on port 25 without requiring it (interoperability).
- Set `smtpd_tls_security_level = encrypt` on the submission port (587) to require TLS for client submissions.
