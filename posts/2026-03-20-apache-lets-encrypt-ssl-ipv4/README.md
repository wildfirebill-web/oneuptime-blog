# How to Set Up Let's Encrypt SSL with Apache on an IPv4 Server

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Apache, Let's Encrypt, SSL, IPv4, Certbot, HTTPS, Security

Description: Learn how to obtain and automatically renew a free Let's Encrypt TLS certificate for an Apache server bound to an IPv4 address.

---

Let's Encrypt provides free, automated TLS certificates via the ACME protocol. Certbot is the official ACME client that integrates directly with Apache to install and renew certificates with minimal manual work.

## Prerequisites

- A domain pointing to your server's IPv4 address in DNS.
- Apache running and reachable on port 80 (for the HTTP-01 challenge).
- Ports 80 and 443 open in your firewall.

## Installing Certbot

```bash
# Debian/Ubuntu

apt update && apt install certbot python3-certbot-apache -y

# RHEL/Rocky/AlmaLinux (via EPEL)
dnf install epel-release -y
dnf install certbot python3-certbot-apache -y
```

## Obtain and Install the Certificate

Certbot's Apache plugin automatically edits your virtual host configuration.

```bash
# Replace example.com with your actual domain
# Certbot will detect the Apache vhost and configure SSL automatically
certbot --apache -d example.com -d www.example.com
```

Certbot will:
1. Verify domain ownership via an HTTP-01 challenge on port 80.
2. Obtain a certificate from Let's Encrypt.
3. Create a new SSL virtual host configuration.
4. Configure automatic HTTP → HTTPS redirection.

## What Certbot Creates

After running Certbot, you'll find an SSL configuration like this:

```apacheconf
# /etc/apache2/sites-available/example.com-le-ssl.conf (auto-generated)
<VirtualHost 0.0.0.0:443>
    ServerName example.com
    ServerAlias www.example.com
    DocumentRoot /var/www/example

    SSLEngine on
    SSLCertificateFile      /etc/letsencrypt/live/example.com/cert.pem
    SSLCertificateKeyFile   /etc/letsencrypt/live/example.com/privkey.pem
    SSLCertificateChainFile /etc/letsencrypt/live/example.com/chain.pem

    Include /etc/letsencrypt/options-ssl-apache.conf
</VirtualHost>
```

## Binding to a Specific IPv4 Address

If you want to restrict the certificate to one IPv4 address, edit the generated config:

```apacheconf
# Change 0.0.0.0:443 to your specific IPv4 address
<VirtualHost 203.0.113.10:443>
    ServerName example.com
    # ... rest of SSL config
</VirtualHost>
```

## Automatic Renewal

Certbot installs a systemd timer (or cron job) that renews certificates before expiry.

```bash
# Test the renewal process (dry run, no changes made)
certbot renew --dry-run

# Check the renewal timer
systemctl status certbot.timer

# Manually list all certificates and their expiry dates
certbot certificates
```

## Forcing HTTPS Redirect

```apacheconf
# /etc/apache2/sites-available/example.com.conf
<VirtualHost 203.0.113.10:80>
    ServerName example.com
    ServerAlias www.example.com
    # Redirect all HTTP traffic to HTTPS permanently
    RewriteEngine On
    RewriteRule ^(.*)$ https://%{HTTP_HOST}$1 [R=301,L]
</VirtualHost>
```

## Key Takeaways

- `certbot --apache` automates certificate installation and Apache configuration.
- Certificates renew automatically via the Certbot systemd timer every 60 days.
- Use `certbot renew --dry-run` to verify renewal works before the actual expiry.
- Edit the generated virtual host to bind to a specific IPv4 address if needed.
