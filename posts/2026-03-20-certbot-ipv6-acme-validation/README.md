# How to Configure certbot for IPv6 ACME Validation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: certbot, IPv6, ACME, Let's Encrypt, TLS, SSL, DevOps

Description: Configure certbot to perform ACME domain validation over IPv6, including binding preferences, challenge type selection, and dual-stack server considerations.

---

certbot, the official Let's Encrypt client, works with IPv6 out of the box in many cases, but correctly configuring it for IPv6 validation requires understanding how ACME challenges work and how certbot binds to network interfaces.

## How ACME Validation Works with IPv6

When Let's Encrypt validates your domain, it performs an HTTP request (for HTTP-01) or DNS lookup (for DNS-01) to verify ownership. For IPv6 validation to succeed:

1. Your domain's DNS must have an AAAA record pointing to your server's IPv6 address.
2. Port 80 (HTTP-01) must be reachable from the ACME server over IPv6.
3. The certbot process or your web server must be listening on the IPv6 interface.

## Checking IPv6 Readiness Before Running certbot

```bash
# Verify AAAA record exists
dig AAAA yourdomain.example.com +short

# Check if port 80 is reachable over IPv6 from the internet
# Run this from an external IPv6-capable host:
curl -6 http://yourdomain.example.com/

# Confirm certbot can bind to IPv6 on the server
python3 -c "import socket; s=socket.socket(socket.AF_INET6, socket.SOCK_STREAM); s.bind(('::', 80)); print('IPv6 bind OK')"
```

## Running certbot in Standalone Mode with IPv6

In standalone mode, certbot starts its own temporary HTTP server. It binds to all interfaces by default, including IPv6:

```bash
# Run certbot standalone - binds to [::]:80 automatically
sudo certbot certonly \
  --standalone \
  --preferred-challenges http-01 \
  --domain yourdomain.example.com \
  --email your@email.com \
  --agree-tos

# Force IPv6 by setting the preferred network interface
# (use --bind-address for acme-tiny, but certbot uses OS default)
```

## Forcing IPv4 or IPv6 Challenge Preference

certbot doesn't have a direct `--ipv6` flag, but you can influence behavior through DNS:

```bash
# If your domain has both A and AAAA records, Let's Encrypt may try both.
# To force IPv6-only validation, remove the A record temporarily,
# or use the pre-hook to verify IPv6 is the active path.

# Check which address Let's Encrypt will use
dig yourdomain.example.com
dig AAAA yourdomain.example.com
```

## Integrating certbot with Nginx for IPv6

Configure Nginx to handle ACME challenges on both IPv4 and IPv6:

```nginx
server {
    # Listen on both protocol versions
    listen 80;
    listen [::]:80;

    server_name yourdomain.example.com;

    # Serve ACME challenge files from this directory
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
        allow all;
    }

    # Redirect all other traffic to HTTPS
    location / {
        return 301 https://$host$request_uri;
    }
}
```

Then run certbot with webroot:

```bash
# Create the webroot directory
sudo mkdir -p /var/www/certbot/.well-known/acme-challenge

# Obtain certificate using webroot method
sudo certbot certonly \
  --webroot \
  --webroot-path /var/www/certbot \
  --domain yourdomain.example.com \
  --email your@email.com \
  --agree-tos
```

## Configuring certbot for IPv6-Only Servers

For IPv6-only servers (no IPv4), DNS-01 is the most reliable challenge type since it requires no inbound connections:

```bash
# Install a DNS plugin matching your provider
# Example for Route53:
pip install certbot-dns-route53

# Configure AWS credentials
mkdir -p ~/.aws
cat > ~/.aws/credentials << 'EOF'
[default]
aws_access_key_id = YOUR_ACCESS_KEY
aws_secret_access_key = YOUR_SECRET_KEY
EOF

# Obtain certificate via DNS-01 (no inbound port 80 needed)
sudo certbot certonly \
  --dns-route53 \
  --domain yourdomain.example.com \
  --email your@email.com \
  --agree-tos
```

## Checking certbot Logs for IPv6 Issues

When validation fails, examine the certbot log:

```bash
# View the most recent certbot log
sudo cat /var/log/letsencrypt/letsencrypt.log | tail -100

# Search for IPv6-specific errors
sudo grep -i "ipv6\|inet6\|connection refused\|timeout" \
  /var/log/letsencrypt/letsencrypt.log
```

## Renewal Hook for IPv6 Servers

Add a renewal hook to ensure your web server is updated after certificate renewal:

```bash
# Create a post-renewal hook
cat > /etc/letsencrypt/renewal-hooks/post/reload-nginx.sh << 'EOF'
#!/bin/bash
# Reload nginx to pick up the renewed certificate
systemctl reload nginx
echo "Nginx reloaded after certificate renewal at $(date)"
EOF

chmod +x /etc/letsencrypt/renewal-hooks/post/reload-nginx.sh
```

certbot's support for IPv6 is mature — with proper DNS configuration and the right challenge type, you can fully automate certificate issuance and renewal on any IPv6-enabled server.
