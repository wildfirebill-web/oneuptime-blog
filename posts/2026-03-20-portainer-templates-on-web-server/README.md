# How to Host Custom Portainer Templates on a Web Server

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Templates, Web Server, DevOps

Description: Learn how to host a custom Portainer template catalog on your own web server for private or internal use.

## Introduction

For organizations that cannot use public platforms like GitHub, hosting Portainer templates on an internal web server is the ideal solution. This approach works well for air-gapped environments, organizations with strict data residency requirements, or when templates reference internal registries and configuration. This guide covers setting up a web server to host your template catalog.

## Prerequisites

- A Linux server accessible from your Portainer instance
- Docker installed on the server
- Basic understanding of Nginx or Caddy
- Portainer CE or BE admin access

## Architecture Overview

```text
Portainer → HTTP GET → Web Server → templates.json
                                  → stacks/*/docker-compose.yml
```

Portainer fetches the `templates.json` file via HTTP/HTTPS. Stack templates reference Compose files that Portainer also fetches directly from the web server or Git repository.

## Option A: Host with Nginx in Docker

### Step 1: Create the Template Directory Structure

```bash
# Create directory structure on the web server

mkdir -p /opt/portainer-templates/stacks/{wordpress,monitoring,gitea}
mkdir -p /opt/portainer-templates/logos
```

### Step 2: Create the Templates JSON

```bash
cat > /opt/portainer-templates/templates.json << 'EOF'
{
  "version": "2",
  "templates": [
    {
      "type": 1,
      "title": "Internal App",
      "description": "Our internal application",
      "categories": ["internal"],
      "platform": "linux",
      "image": "registry.internal.company.com/myapp:latest",
      "ports": ["8080/tcp"],
      "env": [
        {
          "name": "APP_ENV",
          "label": "Environment",
          "default": "production"
        }
      ],
      "restart_policy": "unless-stopped"
    },
    {
      "type": 2,
      "title": "Monitoring Stack",
      "description": "Prometheus and Grafana",
      "categories": ["monitoring"],
      "platform": "linux",
      "repository": {
        "url": "http://templates.internal.company.com",
        "stackfile": "stacks/monitoring/docker-compose.yml"
      },
      "env": [
        {
          "name": "GRAFANA_PASSWORD",
          "label": "Grafana admin password"
        }
      ]
    }
  ]
}
EOF
```

### Step 3: Create a Compose File for a Stack Template

```yaml
# /opt/portainer-templates/stacks/monitoring/docker-compose.yml
version: "3.8"

services:
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - prometheus-data:/prometheus
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
    volumes:
      - grafana-data:/var/lib/grafana
    depends_on:
      - prometheus
    restart: unless-stopped

volumes:
  prometheus-data:
  grafana-data:
```

### Step 4: Deploy Nginx to Serve Templates

```yaml
# /opt/portainer-templates/docker-compose.yml
version: "3.8"

services:
  template-server:
    image: nginx:alpine
    container_name: portainer-template-server
    ports:
      - "8080:80"
    volumes:
      # Serve the templates directory
      - /opt/portainer-templates:/usr/share/nginx/html:ro
      - /opt/portainer-templates/nginx.conf:/etc/nginx/conf.d/default.conf:ro
    restart: unless-stopped
```

```nginx
# /opt/portainer-templates/nginx.conf
server {
    listen 80;
    server_name _;

    root /usr/share/nginx/html;
    index templates.json;

    # Enable CORS for Portainer
    add_header Access-Control-Allow-Origin "*";
    add_header Access-Control-Allow-Methods "GET, OPTIONS";

    # Serve JSON with correct content type
    location ~* \.json$ {
        add_header Content-Type "application/json";
    }

    # Cache static files
    location ~* \.(yml|yaml|json)$ {
        expires 5m;
        add_header Cache-Control "public, no-transform";
    }
}
```

```bash
# Start the template server
docker compose -f /opt/portainer-templates/docker-compose.yml up -d
```

## Option B: Host with Caddy (HTTPS Automatic)

```yaml
# docker-compose.yml for Caddy-based template server
version: "3.8"

services:
  template-server:
    image: caddy:alpine
    container_name: portainer-template-server
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /opt/portainer-templates:/srv:ro
      - caddy-data:/data
      - /opt/portainer-templates/Caddyfile:/etc/caddy/Caddyfile:ro
    restart: unless-stopped

volumes:
  caddy-data:
```

```text
# Caddyfile
templates.company.com {
    root * /srv
    file_server

    # Allow CORS for Portainer
    header Access-Control-Allow-Origin *
    header Content-Type application/json {path}/*.json
}
```

## Option C: Python Simple HTTP Server (Development/Testing)

```bash
# Quick test server - NOT for production
cd /opt/portainer-templates
python3 -m http.server 8080

# Access at: http://server-ip:8080/templates.json
```

## Step 5: Configure Portainer

1. In Portainer, go to **Settings**
2. Set the **App Templates URL**:

```text
http://templates.internal.company.com:8080/templates.json
# or with HTTPS:
https://templates.internal.company.com/templates.json
```

3. Save settings
4. Go to **App Templates** to verify

## Automating Template Updates

Set up a cron job or CI/CD pipeline to update templates automatically:

```bash
#!/bin/bash
# /opt/portainer-templates/update-templates.sh

# Pull latest templates from internal Git
cd /opt/portainer-templates
git pull origin main

# Nginx will serve the updated files immediately (no restart needed)
echo "Templates updated at $(date)"
```

```bash
# Cron job to update every 5 minutes
echo "*/5 * * * * /opt/portainer-templates/update-templates.sh >> /var/log/template-update.log 2>&1" | crontab -
```

## Security Considerations

```nginx
# Restrict access to specific IPs (Portainer server only)
server {
    listen 80;

    allow 10.0.1.100;    # Portainer server IP
    deny all;

    # ... rest of config
}
```

For HTTPS, use a certificate from Let's Encrypt (with Caddy auto-HTTPS) or your internal CA.

## Troubleshooting

```bash
# Test the template URL
curl http://templates.internal.company.com:8080/templates.json | python3 -m json.tool

# Check Nginx logs
docker logs portainer-template-server

# Verify CORS headers
curl -I http://templates.internal.company.com:8080/templates.json
```

## Conclusion

Hosting Portainer templates on your own web server gives you full control over your template catalog. Using Nginx or Caddy in Docker makes it straightforward to set up a reliable, low-maintenance template server. This approach is ideal for organizations with strict security requirements or air-gapped environments where public GitHub access is not available.
