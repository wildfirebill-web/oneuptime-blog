# How to Use Certbot to Secure Portainer with SSL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: portainer, certbot, ssl, tls, acme, security

Description: A detailed guide to using Certbot to obtain and manage SSL certificates for Portainer, including automated renewal and multiple ACME challenge methods.

## Overview

Certbot is the official ACME client for Let's Encrypt. It automates the process of obtaining, installing, and renewing SSL certificates. This guide covers using Certbot specifically for Portainer, including standalone mode, webroot mode, DNS-01 challenges for private networks, and automated renewal hooks.

## Prerequisites

- A domain name (for HTTP-01 challenge, it must resolve to your server)
- Certbot installed
- Portainer running on Docker

## Step 1: Install Certbot

```bash
# Ubuntu 22.04/24.04
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot

# Ubuntu 20.04 / Debian
sudo apt-get install -y certbot

# RHEL/Rocky/Oracle Linux
sudo dnf install -y epel-release
sudo dnf install -y certbot

# Verify
certbot --version
```

## Method 1: Standalone Mode

Certbot runs its own web server on port 80 to prove domain ownership:

```bash
# Portainer must not be on port 80 (it's not by default)
sudo certbot certonly --standalone \
  -d portainer.example.com \
  --agree-tos \
  --non-interactive \
  -m admin@example.com \
  --preferred-challenges http

# Certificates created at:
# /etc/letsencrypt/live/portainer.example.com/
```

## Method 2: Webroot Mode (with Nginx)

If Nginx is running on port 80, use webroot mode instead:

```bash
# Configure Nginx to serve the ACME challenge
sudo mkdir -p /var/www/certbot
sudo tee /etc/nginx/conf.d/acme-challenge.conf << 'EOF'
server {
    listen 80;
    server_name portainer.example.com;
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }
    location / {
        return 301 https://$host$request_uri;
    }
}
EOF

sudo nginx -t && sudo systemctl reload nginx

# Obtain cert using webroot
sudo certbot certonly --webroot \
  -w /var/www/certbot \
  -d portainer.example.com \
  --agree-tos \
  --non-interactive \
  -m admin@example.com
```

## Method 3: DNS-01 Challenge (for Private Networks)

For Portainer instances not reachable from the internet:

```bash
# DNS-01 requires DNS API access
# Example with Cloudflare
sudo pip3 install certbot-dns-cloudflare

# Create Cloudflare credentials
sudo mkdir -p /etc/letsencrypt
sudo tee /etc/letsencrypt/cloudflare.ini << 'EOF'
dns_cloudflare_api_token = your-cloudflare-api-token
EOF
sudo chmod 600 /etc/letsencrypt/cloudflare.ini

# Obtain cert via DNS-01
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /etc/letsencrypt/cloudflare.ini \
  -d portainer.internal.example.com \
  --agree-tos \
  --non-interactive \
  -m admin@example.com
```

## Step 4: Deploy Certbot Certificate to Portainer

```bash
#!/bin/bash
# deploy-cert-to-portainer.sh
DOMAIN="portainer.example.com"
CERT_PATH="/etc/letsencrypt/live/${DOMAIN}"

# Copy certs to Portainer volume
docker run --rm \
  -v portainer_data:/data \
  -v "${CERT_PATH}:/certs:ro" \
  alpine \
  sh -c "mkdir -p /data/certs && \
    cp /certs/fullchain.pem /data/certs/cert.pem && \
    cp /certs/privkey.pem /data/certs/key.pem && \
    chmod 644 /data/certs/cert.pem && \
    chmod 600 /data/certs/key.pem"

# Restart Portainer to pick up new cert
docker restart portainer
echo "Certificate deployed and Portainer restarted"
```

## Step 5: Configure Automatic Renewal with Deploy Hook

```bash
# Create renewal hook
sudo tee /etc/letsencrypt/renewal-hooks/deploy/portainer.sh << 'EOF'
#!/bin/bash
DOMAIN="portainer.example.com"
CERT_PATH="/etc/letsencrypt/live/${DOMAIN}"

docker run --rm \
  -v portainer_data:/data \
  -v "${CERT_PATH}:/certs:ro" \
  alpine \
  sh -c "cp /certs/fullchain.pem /data/certs/cert.pem && cp /certs/privkey.pem /data/certs/key.pem"

docker restart portainer
logger "Portainer: Let's Encrypt certificate renewed and deployed"
EOF

sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/portainer.sh

# Test renewal simulation
sudo certbot renew --dry-run
```

## Monitor Certificate Expiry

```bash
# Check when certificates expire
sudo certbot certificates

# Output:
# Found the following certs:
#   Certificate Name: portainer.example.com
#     Domains: portainer.example.com
#     Expiry Date: 2026-06-18 (VALID: 89 days)
#     Certificate Path: /etc/letsencrypt/live/portainer.example.com/fullchain.pem

# Set up expiry monitoring
echo | openssl s_client -connect portainer.example.com:9443 2>/dev/null \
  | openssl x509 -noout -enddate
```

## Conclusion

Certbot with multiple challenge methods covers Portainer deployments across all network environments — standalone for simple setups, webroot for servers already running Nginx, and DNS-01 for private/internal Portainer instances. Renewal hooks ensure certificates are automatically deployed to Portainer without manual intervention, maintaining continuous HTTPS availability.
