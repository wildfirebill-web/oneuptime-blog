# How to Deploy Nginx Proxy Manager as a Portainer Stack

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Nginx Proxy Manager, Stack, Docker Compose, Deployment

Description: Learn how to deploy Nginx Proxy Manager as a Portainer stack with both SQLite and MariaDB backends, manage it through the Portainer UI, and integrate it with other stacks.

## Introduction

Deploying Nginx Proxy Manager as a Portainer stack brings NPM under Portainer's lifecycle management — you can deploy, update, and monitor NPM through the same interface you use for all other containers. This creates a cohesive infrastructure where Portainer manages everything, including the proxy that sits in front of it.

## Prerequisites

- Portainer CE or BE running
- Admin access to Portainer
- Domain names pointing to your server (for NPM SSL provisioning)

## Step 1: Create the NPM Stack in Portainer

Navigate to **Stacks** → **Add Stack** in Portainer.

Name the stack: `nginx-proxy-manager`

**Option A: SQLite Backend (Simpler — recommended for single-host)**

```yaml
version: "3.8"

services:
  npm:
    image: jc21/nginx-proxy-manager:latest
    container_name: nginx-proxy-manager
    restart: unless-stopped
    ports:
      - "80:80"      # HTTP
      - "443:443"    # HTTPS
      - "81:81"      # NPM admin UI
    volumes:
      - npm_data:/data
      - npm_letsencrypt:/etc/letsencrypt
    networks:
      - proxy
    healthcheck:
      test: ["CMD", "/bin/check-health"]
      interval: 30s
      timeout: 10s
      retries: 3

networks:
  proxy:
    name: proxy
    driver: bridge

volumes:
  npm_data:
    driver: local
  npm_letsencrypt:
    driver: local
```

## Step 2: MariaDB Backend Stack (Production Recommended)

**Option B: MariaDB Backend with Portainer environment variables**

In Portainer's Stack editor, use the **Environment Variables** section:

```yaml
version: "3.8"

services:
  npm:
    image: jc21/nginx-proxy-manager:latest
    container_name: nginx-proxy-manager
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "81:81"
    environment:
      DB_MYSQL_HOST: "npm-db"
      DB_MYSQL_PORT: 3306
      DB_MYSQL_USER: "${DB_USER}"          # Portainer env var
      DB_MYSQL_PASSWORD: "${DB_PASSWORD}"  # Portainer env var
      DB_MYSQL_NAME: "npm"
    volumes:
      - npm_data:/data
      - npm_letsencrypt:/etc/letsencrypt
    networks:
      - npm-internal
      - proxy
    depends_on:
      npm-db:
        condition: service_healthy

  npm-db:
    image: jc21/mariadb-aria:latest
    container_name: npm-mariadb
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: "${DB_ROOT_PASSWORD}"
      MYSQL_DATABASE: "npm"
      MYSQL_USER: "${DB_USER}"
      MYSQL_PASSWORD: "${DB_PASSWORD}"
    volumes:
      - npm_db_data:/var/lib/mysql
    networks:
      - npm-internal
    healthcheck:
      test: ["CMD", "healthcheck.sh", "--connect", "--innodb_initialized"]
      interval: 30s
      timeout: 10s
      retries: 3

networks:
  npm-internal:
    driver: bridge
  proxy:
    name: proxy
    driver: bridge

volumes:
  npm_data:
  npm_letsencrypt:
  npm_db_data:
```

Set Portainer environment variables:
- `DB_USER`: npm
- `DB_PASSWORD`: your-secure-db-password
- `DB_ROOT_PASSWORD`: your-secure-root-password

## Step 3: Configure the Proxy Network for Other Stacks

Other stacks that need NPM to proxy them should join the `proxy` network:

```yaml
# In any other Portainer stack
version: "3.8"

services:
  myapp:
    image: myapp:latest
    networks:
      - proxy    # Joins the proxy network — NPM can forward to it

networks:
  proxy:
    external: true    # Reference the existing proxy network
    name: proxy
```

## Step 4: Stack Management via Portainer

After deploying, Portainer shows the NPM stack in the Stacks list with:

```
Stack: nginx-proxy-manager
Services: npm (running), npm-db (running)
Status: Running

Actions available:
  Start/Stop — Bring NPM down for maintenance
  Update — Pull new image version
  Edit — Modify compose file inline
  Logs — View NPM and MariaDB logs
  Console — Access container shell for debugging
```

## Step 5: Update NPM Through Portainer

```bash
# Method 1: Through Portainer UI
# Go to Stacks → nginx-proxy-manager → Edit
# Change: jc21/nginx-proxy-manager:latest → jc21/nginx-proxy-manager:2.11.3
# Click Update the stack
# Enable "Re-pull image" checkbox

# Method 2: Via Portainer API
PORTAINER_URL="https://portainer.example.com"
TOKEN="your-admin-token"

# Get stack ID
STACK_ID=$(curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/stacks" | \
  jq -r '.[] | select(.Name == "nginx-proxy-manager") | .Id')

# Trigger stack redeploy with image pull
curl -s -X PUT -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "${PORTAINER_URL}/api/stacks/${STACK_ID}/git/redeploy" \
  -d '{"pullImage": true}'
```

## Step 6: Backup NPM Configuration via Portainer

```bash
# Access NPM container console through Portainer
# Portainer → Containers → nginx-proxy-manager → Console

# Inside the container, export configuration
sqlite3 /data/database.sqlite .dump > /data/npm-backup-$(date +%Y%m%d).sql

# Or from the host using Portainer's exec API
curl -s -X POST -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/1/docker/containers/nginx-proxy-manager/exec" \
  -d '{"Cmd": ["sqlite3", "/data/database.sqlite", ".dump"], "AttachStdout": true}'
```

## Conclusion

Deploying Nginx Proxy Manager as a Portainer stack brings all lifecycle management under a single interface. The SQLite backend is appropriate for single-host deployments while the MariaDB option provides better reliability for production. Use Portainer's environment variables to keep database credentials out of the compose file, and create a shared `proxy` network as an external resource so all managed stacks can be reached by NPM without being part of the NPM stack definition.
