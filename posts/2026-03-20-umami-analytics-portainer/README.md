# How to Deploy Umami Analytics via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Umami, Analytics, Self-Hosted, Privacy

Description: Deploy Umami, the simple and fast privacy-focused analytics platform, as a Docker stack through Portainer for lightweight website traffic tracking.

## Introduction

Umami is a minimalist, open-source web analytics tool that provides a clean, fast dashboard with no cookies and no personal data collection. It's an excellent lightweight alternative to Plausible or Matomo, and can be deployed in minutes via a Portainer stack.

## Prerequisites

- Portainer CE or BE installed
- Docker Engine 20.10+
- 512 MB RAM minimum (Umami is very lightweight)

## Step 1: Generate a Secret

```bash
# Generate a random secret for JWT tokens
openssl rand -hex 32
```

## Step 2: Create the Stack in Portainer

Navigate to **Stacks** → **Add Stack** → **Web Editor**:

```yaml
version: "3.8"

services:
  # PostgreSQL database for Umami
  umami-db:
    image: postgres:15-alpine
    container_name: umami-db
    restart: unless-stopped
    environment:
      POSTGRES_DB: umami
      POSTGRES_USER: umami
      POSTGRES_PASSWORD: umamipassword
    volumes:
      - umami_db_data:/var/lib/postgresql/data
    networks:
      - umami-net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U umami -d umami"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Umami Analytics application
  umami:
    image: ghcr.io/umami-software/umami:postgresql-latest
    container_name: umami
    restart: unless-stopped
    depends_on:
      umami-db:
        condition: service_healthy
    ports:
      - "3000:3000"
    environment:
      # Database connection string
      DATABASE_URL: postgresql://umami:umamipassword@umami-db:5432/umami

      # JWT secret — generate with 'openssl rand -hex 32'
      APP_SECRET: "your_generated_hex_secret_here"

      # Optional: disable telemetry
      DISABLE_TELEMETRY: "1"

      # Optional: set a custom base path
      # BASE_PATH: /analytics
    networks:
      - umami-net

volumes:
  umami_db_data:

networks:
  umami-net:
    driver: bridge
```

### Using MySQL Instead of PostgreSQL

```yaml
  umami-db:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: umami
      MYSQL_USER: umami
      MYSQL_PASSWORD: umamipassword
      MYSQL_ROOT_PASSWORD: rootpassword

  umami:
    image: ghcr.io/umami-software/umami:mysql-latest
    environment:
      DATABASE_URL: mysql://umami:umamipassword@umami-db:3306/umami
```

## Step 3: Deploy the Stack

1. Name the stack `umami`
2. Click **Deploy the stack**
3. Wait about 30 seconds for the database to initialize

## Step 4: Log In to Umami

1. Open `http://your-host:3000`
2. Default credentials: `admin` / `umami`
3. **Immediately change the default password** under **Profile** → **Change Password**

## Step 5: Add Your First Website

1. Go to **Settings** → **Websites**
2. Click **Add Website**
3. Enter your website name and domain
4. Umami generates a tracking script

## Step 6: Add the Tracking Script

```html
<!-- Add to your website's <head> section -->
<script
  async
  src="https://analytics.yourdomain.com/script.js"
  data-website-id="your-website-uuid-here">
</script>
```

The website UUID is shown in Umami's website settings.

## Step 7: Using Umami's Tracking API

For custom events:

```javascript
// Track a custom event (Umami v2 API)
umami.track('button-click', { button: 'signup', page: 'home' });

// Track a page view manually (for SPAs)
umami.track({ url: '/new-page', title: 'New Page Title' });
```

## Step 8: Configure Reverse Proxy (Traefik)

Add labels to the Umami service for Traefik:

```yaml
umami:
  labels:
    - "traefik.enable=true"
    - "traefik.http.routers.umami.rule=Host(`analytics.yourdomain.com`)"
    - "traefik.http.routers.umami.tls=true"
    - "traefik.http.routers.umami.tls.certresolver=letsencrypt"
    - "traefik.http.services.umami.loadbalancer.server.port=3000"
```

## Step 9: Share Analytics Publicly

Umami allows you to make dashboards public:

1. Go to **Settings** → **Websites** → select your site
2. Toggle **Enable share URL**
3. Share the generated URL for public view access

## Step 10: Multi-User Setup

Create team members:

1. **Settings** → **Users** → **Add User**
2. Assign the **admin** or **user** role
3. Users can only see websites they're assigned to (unless admin)

## Conclusion

Umami is one of the lightest and fastest self-hosted analytics tools available. Deployed via Portainer, you get a privacy-respecting analytics solution that uses minimal resources, stores all data in your own PostgreSQL database, and provides a beautiful, fast dashboard — all without a single cookie.
