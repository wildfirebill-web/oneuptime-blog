# How to Configure Caddy Automatic HTTPS with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Caddy, HTTPS, IPv6, TLS, Let's Encrypt, Web Server, Automatic Certificates

Description: Configure the Caddy web server to automatically obtain and renew TLS certificates on IPv6-enabled servers, leveraging Caddy's built-in ACME client and IPv6 support.

---

Caddy is known for making HTTPS automatic and effortless. It has excellent IPv6 support out of the box, binding to both IPv4 and IPv6 by default and using its built-in ACME client (using the `certmagic` library) to obtain Let's Encrypt certificates.

## Installing Caddy

```bash
# Ubuntu/Debian

sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' \
  | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' \
  | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update && sudo apt install caddy

# Verify installation and version
caddy version
```

## Basic Caddyfile for IPv6 Auto-HTTPS

Caddy's Caddyfile format is concise. IPv6 listening happens automatically:

```caddy
# /etc/caddy/Caddyfile

# Caddy binds to both [::]:443 and 0.0.0.0:443 by default
example.com {
    # Caddy automatically fetches and renews Let's Encrypt certificate
    reverse_proxy localhost:8080
}

# To explicitly listen on IPv6 only:
http://[::]:80 {
    redir https://{host}{uri} permanent
}
```

## Binding Caddy to IPv6 Addresses

For explicit IPv6 control in the Caddyfile:

```caddy
# /etc/caddy/Caddyfile

# Bind specifically to IPv6 address
[2001:db8::1]:443 {
    tls /etc/ssl/certs/server.crt /etc/ssl/private/server.key
    respond "Hello from IPv6!"
}

# Dual-stack with automatic HTTPS
example.com {
    # Caddy handles HTTPS on both IPv4 and IPv6 automatically
    encode gzip
    file_server {
        root /var/www/html
    }
}
```

## Using DNS-01 Challenge for IPv6-Only Servers

For IPv6-only servers or wildcard certificates, configure DNS-01 in the Caddyfile:

```bash
# Install the Cloudflare DNS plugin for Caddy
# Download the xcaddy builder
go install github.com/caddyserver/xcaddy/cmd/xcaddy@latest

# Build Caddy with Cloudflare DNS plugin
xcaddy build --with github.com/caddy-dns/cloudflare
```

```caddy
# /etc/caddy/Caddyfile

{
    # Global ACME settings
    acme_dns cloudflare {env.CF_API_TOKEN}
    email admin@example.com
}

# Wildcard certificate via DNS-01
*.example.com {
    reverse_proxy localhost:8080
}

example.com {
    redir https://www.example.com{uri}
}
```

Set the environment variable:

```bash
# Add to /etc/caddy/caddy.env or systemd override
CF_API_TOKEN=your_cloudflare_api_token

# Or create a systemd override
sudo systemctl edit caddy
```

```ini
# Systemd override for Caddy
[Service]
Environment="CF_API_TOKEN=your_cloudflare_api_token"
```

## Configuring ACME Server and Email

```caddy
# /etc/caddy/Caddyfile

{
    # Use Let's Encrypt production (default)
    acme_ca https://acme-v02.api.letsencrypt.org/directory

    # Or use staging for testing
    # acme_ca https://acme-staging-v02.api.letsencrypt.org/directory

    email admin@example.com

    # Configure HTTP-01 challenge port (if non-standard)
    http_port 80
    https_port 443
}

example.com {
    respond "IPv6 HTTPS works!"
}
```

## Testing Caddy IPv6 Configuration

```bash
# Validate the Caddyfile
caddy validate --config /etc/caddy/Caddyfile

# Start/reload Caddy
sudo systemctl restart caddy
sudo systemctl status caddy

# Test IPv6 connectivity
curl -6 https://example.com
openssl s_client -connect '[2001:db8::1]:443' -servername example.com < /dev/null

# Check Caddy's certificate storage
sudo ls /var/lib/caddy/.local/share/caddy/certificates/
```

## Checking Caddy Logs for IPv6 Issues

```bash
# View Caddy logs
sudo journalctl -u caddy -f

# Look for ACME and IPv6 specific messages
sudo journalctl -u caddy | grep -i "acme\|ipv6\|certificate\|error"

# Caddy admin API for certificate status
curl http://localhost:2019/config/
curl http://localhost:2019/pki/ca/local
```

## Caddy JSON Config Alternative

For programmatic configuration, use JSON format:

```json
{
  "apps": {
    "http": {
      "servers": {
        "myserver": {
          "listen": ["[::]:443"],
          "routes": [
            {
              "match": [{"host": ["example.com"]}],
              "handle": [{"handler": "static_response", "body": "Hello IPv6!"}]
            }
          ]
        }
      }
    },
    "tls": {
      "automation": {
        "policies": [
          {
            "subjects": ["example.com"],
            "issuers": [{"module": "acme"}]
          }
        ]
      }
    }
  }
}
```

Caddy's automatic HTTPS combined with its native IPv6 support makes it one of the easiest web servers to deploy on modern dual-stack and IPv6-only infrastructure.
