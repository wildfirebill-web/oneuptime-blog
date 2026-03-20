# How to Use Nginx Proxy Manager to Forward Traffic in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Nginx Proxy Manager, Portainer, Docker, Reverse Proxy, SSL

Description: Learn how to deploy Nginx Proxy Manager alongside Portainer and use it to forward traffic to containerized services with automatic SSL certificate management.

## Introduction

Nginx Proxy Manager (NPM) is a user-friendly reverse proxy with a web-based GUI for managing proxy hosts, SSL certificates, and redirections. Deploying NPM through Portainer and using it to forward traffic to your services eliminates the need for complex Nginx configuration files.

## Prerequisites

- Portainer installed and running
- A domain name with DNS pointing to your server
- Ports 80 and 443 open on your firewall

## Step 1: Deploy Nginx Proxy Manager via Portainer Stack

In Portainer, navigate to **Stacks** > **Add stack** and paste:

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
      - "81:81"    # Admin UI
    volumes:
      - npm_data:/data
      - npm_letsencrypt:/etc/letsencrypt
    environment:
      DB_MYSQL_HOST: "npm-db"
      DB_MYSQL_PORT: 3306
      DB_MYSQL_USER: "npm"
      DB_MYSQL_PASSWORD: "${DB_PASSWORD}"
      DB_MYSQL_NAME: "npm"

  npm-db:
    image: mariadb:10.11
    container_name: npm-db
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: "${DB_ROOT_PASSWORD}"
      MYSQL_DATABASE: "npm"
      MYSQL_USER: "npm"
      MYSQL_PASSWORD: "${DB_PASSWORD}"
    volumes:
      - npm_db:/var/lib/mysql

volumes:
  npm_data:
  npm_letsencrypt:
  npm_db:
```

Set the environment variables in Portainer's stack editor before deploying.

## Step 2: Access the NPM Admin UI

Open your browser to `http://your-server-ip:81`.

Default credentials:
- Email: `admin@example.com`
- Password: `changeme`

Change these immediately after first login.

## Step 3: Add a Proxy Host

1. In NPM, click **Proxy Hosts** > **Add Proxy Host**.
2. Fill in the details:
   - **Domain Names**: `myapp.example.com`
   - **Scheme**: `http`
   - **Forward Hostname/IP**: Container name (e.g., `myapp`) or IP
   - **Forward Port**: Container's port (e.g., `3000`)
   - Enable **Block Common Exploits**

3. On the **SSL** tab:
   - Select **Request a new SSL Certificate**
   - Enable **Force SSL**
   - Enable **HTTP/2 Support**
   - Enter your email for Let's Encrypt
   - Check the **I Agree** checkbox

4. Click **Save**. NPM will provision a Let's Encrypt certificate automatically.

## Step 4: Deploy the Application Stack in Portainer

The app doesn't need to expose ports to the host - NPM handles external access:

```yaml
version: "3.8"

services:
  myapp:
    image: myapp:latest
    container_name: myapp
    restart: unless-stopped
    environment:
      NODE_ENV: production
      DATABASE_URL: postgres://db:5432/myapp
    # No ports needed - NPM forwards to the container by name
    networks:
      - proxy

  db:
    image: postgres:15
    restart: unless-stopped
    environment:
      POSTGRES_DB: myapp
      POSTGRES_PASSWORD: "${DB_PASSWORD}"
    volumes:
      - app_db:/var/lib/postgresql/data
    networks:
      - proxy

volumes:
  app_db:

networks:
  proxy:
    external: true
```

## Step 5: Create a Shared Network

Both NPM and your app stacks need to share a Docker network:

```bash
docker network create proxy
```

Or in Portainer, go to **Networks** > **Add network** > name it `proxy`.

Update your NPM stack to use this network:

```yaml
services:
  npm:
    # ... existing config ...
    networks:
      - proxy

networks:
  proxy:
    external: true
```

## Step 6: Add Custom Locations

Route different paths to different containers:

In NPM, edit the proxy host and go to **Custom Locations**:
- Location: `/api`
- Scheme: `http`
- Forward Hostname: `api-container`
- Forward Port: `8080`

## Step 7: Set Up Access Lists

Restrict access to the Portainer UI through NPM:

1. Go to **Access Lists** > **Add Access List**.
2. Add allowed IPs or set basic auth credentials.
3. Apply the access list to the Portainer proxy host.

## Best Practices

- Use Let's Encrypt certificates via NPM for automatic renewal.
- Connect NPM and all app containers to a shared Docker network.
- Do not expose application container ports to the host - route everything through NPM.
- Enable "Block Common Exploits" on all proxy hosts.
- Use access lists to restrict the NPM admin panel to trusted IPs.

## Conclusion

Nginx Proxy Manager deployed through Portainer provides a powerful, GUI-driven reverse proxy solution. By routing all external traffic through NPM with automatic SSL, you eliminate manual certificate management and keep your application containers cleanly isolated from direct internet exposure.
