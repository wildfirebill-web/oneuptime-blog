# How to Deploy Nginx Proxy Manager as a Portainer Stack - Part 2

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Nginx Proxy Manager, Stack, Docker Compose, Reverse Proxy

Description: Learn how to deploy Nginx Proxy Manager as a managed Portainer stack, giving you version-controlled configuration and easy stack updates through the Portainer UI.

## Why Deploy NPM as a Portainer Stack?

Managing NPM as a Portainer stack lets you:
- Track the compose configuration in Portainer's stack history
- Update NPM version from the Portainer UI
- Restart services with one click
- View NPM logs alongside other stacks

## Step 1: Open Portainer Stack Editor

1. Go to **Stacks → Add Stack**
2. Name the stack: `nginx-proxy-manager`
3. Paste the following compose content

## Docker Compose for NPM

```yaml
version: "3.8"

services:
  npm:
    image: jc21/nginx-proxy-manager:latest
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "81:81"      # NPM admin UI
    environment:
      DB_MYSQL_HOST: "npm-db"
      DB_MYSQL_PORT: "3306"
      DB_MYSQL_USER: "npm"
      DB_MYSQL_PASSWORD: "${DB_PASSWORD}"
      DB_MYSQL_NAME: "npm"
      # Optional: disable IPv6 if not needed
      DISABLE_IPV6: "true"
    volumes:
      - npm_data:/data
      - npm_letsencrypt:/etc/letsencrypt
    networks:
      - npm-internal
      - proxy        # Shared with other containers
    depends_on:
      npm-db:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:81"]
      interval: 30s
      timeout: 10s
      retries: 3

  npm-db:
    image: jc21/mariadb-aria:latest
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: "${DB_ROOT_PASSWORD}"
      MYSQL_DATABASE: "npm"
      MYSQL_USER: "npm"
      MYSQL_PASSWORD: "${DB_PASSWORD}"
    volumes:
      - npm_db_data:/var/lib/mysql
    networks:
      - npm-internal
    healthcheck:
      test: ["CMD", "healthcheck.sh", "--su-mysql", "--connect", "--innodb_initialized"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  npm_data:
  npm_letsencrypt:
  npm_db_data:

networks:
  npm-internal:
    driver: bridge
  proxy:
    name: proxy
    external: true
```

## Step 2: Set Environment Variables in Portainer

In the Portainer stack editor, scroll down to **Environment Variables** and add:

```text
DB_PASSWORD=a-strong-database-password
DB_ROOT_PASSWORD=an-even-stronger-root-password
```

Or use Portainer's **Environment** feature to define these in the stack's environment file.

## Step 3: Create the External Proxy Network

Before deploying the stack, create the shared proxy network:

```bash
docker network create proxy
```

Or in Portainer: **Networks → Add Network → Name: proxy, Driver: bridge**

## Step 4: Deploy the Stack

Click **Deploy the Stack**. Portainer creates all services, volumes, and connects to the network.

Monitor the deployment:

```text
Stacks → nginx-proxy-manager → Logs
```

## Step 5: Access NPM Admin

Once running:
- NPM Admin: `http://server:81`
- Default credentials: `admin@example.com` / `changeme`
- Change both immediately after first login

## Step 6: Updating NPM

When a new version of NPM is released:

1. Go to **Stacks → nginx-proxy-manager**
2. Click **Editor**
3. Update `jc21/nginx-proxy-manager:latest` to a specific version tag (e.g., `2.12.1`)
4. Click **Update the Stack**

Portainer pulls the new image and recreates the NPM container while preserving data volumes.

## Backup Strategy

```bash
# Backup volumes

docker run --rm \
  -v npm_data:/npm_data:ro \
  -v npm_letsencrypt:/npm_letsencrypt:ro \
  -v /backup:/backup \
  alpine sh -c "tar czf /backup/npm-backup-$(date +%Y%m%d).tar.gz /npm_data /npm_letsencrypt"
```

## Conclusion

Deploying NPM as a Portainer stack centralizes your reverse proxy management within the same tool you use for all your containers. You get stack-level restarts, updates, and log visibility, while NPM handles SSL and traffic routing for your entire Docker environment.
