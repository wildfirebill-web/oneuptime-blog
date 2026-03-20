# How to Install Portainer Using Docker Compose

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: portainer, docker-compose, installation, docker, infrastructure-as-code

Description: A guide to installing Portainer CE and Portainer Business Edition using Docker Compose for a reproducible, version-controlled deployment.

## Overview

Using Docker Compose to deploy Portainer provides a reproducible, version-controlled deployment that can be easily shared and managed as code. This guide covers Docker Compose configurations for Portainer CE, including HTTPS with custom certificates and environment-specific customizations.

## Prerequisites

- Docker Engine installed
- Docker Compose v2 (included with Docker Desktop and docker-compose-plugin)
- Basic Docker Compose knowledge

## Basic Portainer CE Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: always
    ports:
      - "8000:8000"     # Portainer agent port
      - "9443:9443"     # HTTPS UI
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data

volumes:
  portainer_data:
    driver: local
```

```bash
# Deploy with Docker Compose
docker compose up -d

# View logs
docker compose logs -f portainer

# Check status
docker compose ps
```

## Portainer CE with HTTP Enabled

```yaml
# docker-compose.yml - with HTTP enabled
version: '3.8'

services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: always
    command:
      - --http-enabled    # Enable HTTP port 9000
    ports:
      - "8000:8000"
      - "9000:9000"     # HTTP
      - "9443:9443"     # HTTPS
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data

volumes:
  portainer_data:
```

## Portainer with Custom TLS Certificates

```yaml
# docker-compose.yml - with custom certificates
version: '3.8'

services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: always
    command:
      - --sslcert=/certs/portainer.crt
      - --sslkey=/certs/portainer.key
    ports:
      - "8000:8000"
      - "9443:9443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
      - ./certs:/certs:ro     # Mount certificate directory

volumes:
  portainer_data:
```

```bash
# Create certificate directory and generate self-signed cert
mkdir -p ./certs

openssl req -x509 -newkey rsa:4096 \
  -keyout ./certs/portainer.key \
  -out ./certs/portainer.crt \
  -days 365 \
  -nodes \
  -subj "/CN=portainer.example.com" \
  -addext "subjectAltName=DNS:portainer.example.com,IP:192.168.1.100"

# Deploy
docker compose up -d
```

## Portainer with Nginx Reverse Proxy

```yaml
# docker-compose.yml - Portainer behind Nginx
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    container_name: nginx-proxy
    restart: always
    ports:
      - "443:443"
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/certs:/etc/nginx/certs:ro
    depends_on:
      - portainer

  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: always
    expose:
      - "9443"     # Not exposed publicly - accessed via Nginx
    ports:
      - "8000:8000"   # Agent port still needs to be direct
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data

volumes:
  portainer_data:
```

```nginx
# nginx/nginx.conf
events {}
http {
    server {
        listen 443 ssl;
        server_name portainer.example.com;

        ssl_certificate /etc/nginx/certs/cert.pem;
        ssl_certificate_key /etc/nginx/certs/key.pem;
        ssl_protocols TLSv1.2 TLSv1.3;

        location / {
            proxy_pass https://portainer:9443;
            proxy_ssl_verify off;    # Trust self-signed cert from Portainer
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
    server {
        listen 80;
        return 301 https://$host$request_uri;
    }
}
```

## Portainer CE with Environment Variables

```yaml
# docker-compose.yml with environment variables
version: '3.8'

services:
  portainer:
    image: portainer/portainer-ce:${PORTAINER_VERSION:-latest}
    container_name: portainer
    restart: always
    ports:
      - "${PORTAINER_HTTPS_PORT:-9443}:9443"
      - "${PORTAINER_AGENT_PORT:-8000}:8000"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data

volumes:
  portainer_data:
```

```bash
# .env file
PORTAINER_VERSION=2.20.2
PORTAINER_HTTPS_PORT=9443
PORTAINER_AGENT_PORT=8000
```

## Managing the Deployment

```bash
# Start Portainer
docker compose up -d

# Stop Portainer
docker compose down

# Update to latest version
docker compose pull
docker compose up -d

# View logs
docker compose logs portainer

# Restart
docker compose restart portainer
```

## Conclusion

Using Docker Compose for Portainer deployments provides reproducibility, version control, and easy management. You can commit your docker-compose.yml to Git to track configuration changes. The compose file format is readable and can be shared across team members for consistent deployments. For production environments, store sensitive configuration like certificate paths in environment variables and use `.env` files that are excluded from version control.
