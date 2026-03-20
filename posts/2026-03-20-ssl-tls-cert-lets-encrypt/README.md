# How to Generate and Install an SSL/TLS Certificate Using Let's Encrypt

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Let's Encrypt, SSL, TLS, Certbot, HTTPS, Web Security

Description: Learn how to use Certbot to obtain and install a free SSL/TLS certificate from Let's Encrypt for Nginx or Apache, including automatic renewal setup.

## Why Let's Encrypt?

Let's Encrypt is a free, automated, and open Certificate Authority that issues 90-day SSL/TLS certificates. Combined with Certbot (the official ACME client), the entire process-issuance, installation, and renewal-is fully automated.

## Step 1: Install Certbot

```bash
# Ubuntu/Debian with Snap (recommended)

sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot

# Or via apt (older method)
sudo apt-get install -y certbot python3-certbot-nginx python3-certbot-apache
```

## Step 2: Obtain a Certificate for Nginx

Certbot can automatically configure Nginx:

```bash
# Obtain certificate and auto-configure Nginx
sudo certbot --nginx -d example.com -d www.example.com

# Interactive mode - you'll be prompted for:
# - Email address (for expiry notifications)
# - Agree to terms of service
# - Share email with EFF (optional)
# - Redirect HTTP to HTTPS (choose option 2 for HTTPS redirect)
```

Certbot will:
1. Create a temporary ACME challenge file
2. Request the certificate from Let's Encrypt
3. Automatically update your Nginx configuration
4. Set up automatic renewal

## Step 3: Obtain a Certificate for Apache

```bash
# Auto-configure Apache
sudo certbot --apache -d example.com -d www.example.com
```

## Step 4: Obtain Certificate Only (Manual Installation)

For custom web server configurations:

```bash
# Get certificate without modifying web server config
sudo certbot certonly --webroot \
  -w /var/www/html \
  -d example.com \
  -d www.example.com \
  --email admin@example.com \
  --agree-tos \
  --no-eff-email

# Certificate files are saved to:
# /etc/letsencrypt/live/example.com/fullchain.pem  <- cert + chain
# /etc/letsencrypt/live/example.com/privkey.pem    <- private key
# /etc/letsencrypt/live/example.com/cert.pem       <- cert only
# /etc/letsencrypt/live/example.com/chain.pem      <- chain only
```

## Step 5: Manual Installation in Nginx

Reference the Let's Encrypt certificate files in Nginx:

```nginx
# /etc/nginx/conf.d/example.com.conf
server {
    listen 443 ssl;
    server_name example.com www.example.com;

    # Let's Encrypt certificate files
    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # Use Certbot's recommended SSL settings
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    root /var/www/html;
}
```

## Step 6: Verify Automatic Renewal

Let's Encrypt certificates expire after 90 days. Certbot sets up a cron job or systemd timer for automatic renewal:

```bash
# Verify the renewal timer is active
sudo systemctl status snap.certbot.renew.timer
# or
sudo systemctl status certbot.timer

# Test renewal process (dry run - no actual renewal)
sudo certbot renew --dry-run

# Manual renewal (if needed before expiry)
sudo certbot renew

# Check certificate expiry
sudo certbot certificates

# Output:
# Found the following certs:
#   Certificate Name: example.com
#   Domains: example.com www.example.com
#   Expiry Date: 2026-06-19 10:00:00+00:00 (VALID: 89 days)
```

## Step 7: Obtain Wildcard Certificate

For `*.example.com`, use DNS-01 challenge:

```bash
# Wildcard certificates require DNS challenge
sudo certbot certonly \
  --manual \
  --preferred-challenges dns \
  -d example.com \
  -d "*.example.com"

# You'll be prompted to add a TXT record to your DNS:
# _acme-challenge.example.com TXT "randomvalue123..."

# Or use a DNS plugin for automated DNS challenge:
# For Cloudflare:
sudo apt-get install python3-certbot-dns-cloudflare
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /etc/letsencrypt/cloudflare.ini \
  -d "*.example.com" \
  -d example.com
```

## Step 8: Monitor Certificate Expiry

Set up alerts before the certificate expires:

```bash
# Check days remaining
openssl x509 -enddate -noout -in /etc/letsencrypt/live/example.com/cert.pem

# Script to alert if < 30 days remaining
#!/bin/bash
CERT="/etc/letsencrypt/live/example.com/cert.pem"
DAYS=$(openssl x509 -enddate -noout -in "$CERT" | \
  sed 's/notAfter=//' | \
  xargs -I{} date -d {} +%s | \
  awk -v now="$(date +%s)" '{print int(($1-now)/86400)}')

if [ "$DAYS" -lt 30 ]; then
  echo "WARNING: Certificate expires in ${DAYS} days!"
fi
```

## Conclusion

Let's Encrypt with Certbot provides free, automated SSL/TLS certificates with zero ongoing manual effort. Use `certbot --nginx` or `certbot --apache` for automatic web server configuration, verify automatic renewal with `certbot renew --dry-run`, and monitor expiry dates to catch any renewal failures before they cause outages.
