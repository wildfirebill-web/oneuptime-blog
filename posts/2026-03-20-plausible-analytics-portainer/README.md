# How to Deploy Plausible Analytics via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Plausible Analytics, Self-Hosted, Privacy, Analytics

Description: Deploy Plausible Analytics, the privacy-focused web analytics platform, as a Docker stack through Portainer for GDPR-compliant website traffic analysis.

## Introduction

Plausible Analytics is a lightweight, open-source, privacy-focused alternative to Google Analytics. It doesn't use cookies, is GDPR-compliant out of the box, and can be fully self-hosted. Deploying it via Portainer gives you a manageable, observable deployment on your own infrastructure.

## Prerequisites

- Portainer CE or BE installed
- A domain name pointing to your server (for HTTPS)
- Docker Engine 20.10+
- SMTP credentials for email (optional but recommended)

## Step 1: Prepare Environment Variables

Plausible requires a secret key base. Generate one:

```bash
# Generate a 64-character secret key

openssl rand -base64 64 | tr -d '\n'
```

## Step 2: Create the Stack in Portainer

Navigate to **Stacks** → **Add Stack** → **Web Editor**:

```yaml
version: "3.8"

services:
  # PostgreSQL - Plausible's primary database
  plausible_db:
    image: postgres:16-alpine
    container_name: plausible-db
    restart: unless-stopped
    volumes:
      - plausible_db_data:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: plausible
      POSTGRES_USER: plausible
      POSTGRES_PASSWORD: plausiblepassword
    networks:
      - plausible-net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U plausible"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ClickHouse - Plausible's analytics event store
  plausible_events_db:
    image: clickhouse/clickhouse-server:24.3.3.102-alpine
    container_name: plausible-events-db
    restart: unless-stopped
    volumes:
      - plausible_events_data:/var/lib/clickhouse
      - plausible_events_logs:/var/log/clickhouse-server
    ulimits:
      nofile:
        soft: 262144
        hard: 262144
    networks:
      - plausible-net

  # Plausible Analytics application
  plausible:
    image: ghcr.io/plausible/community-edition:v2.1.4
    container_name: plausible
    restart: unless-stopped
    depends_on:
      plausible_db:
        condition: service_healthy
      plausible_events_db:
        condition: service_started
    command: sh -c "sleep 10 && /entrypoint.sh db createdb && /entrypoint.sh db migrate && /entrypoint.sh run"
    ports:
      - "8000:8000"
    environment:
      # Required: base URL for your Plausible instance
      BASE_URL: https://plausible.yourdomain.com

      # Required: generate with 'openssl rand -base64 64'
      SECRET_KEY_BASE: "your_64_char_secret_key_here"

      # Database connections
      DATABASE_URL: postgres://plausible:plausiblepassword@plausible_db:5432/plausible
      CLICKHOUSE_DATABASE_URL: http://plausible_events_db:8123/plausible_events

      # Optional: SMTP for email invites and reports
      # MAILER_EMAIL: hello@yourdomain.com
      # SMTP_HOST_ADDR: smtp.yourdomain.com
      # SMTP_HOST_PORT: "587"
      # SMTP_USER_NAME: yoursmtpuser
      # SMTP_USER_PWD: yoursmtppassword

      # Disable registration after initial setup
      # DISABLE_REGISTRATION: "true"
    networks:
      - plausible-net

volumes:
  plausible_db_data:
  plausible_events_data:
  plausible_events_logs:

networks:
  plausible-net:
    driver: bridge
```

## Step 3: Deploy the Stack

1. Name the stack `plausible`
2. Click **Deploy the stack**
3. Watch the logs - the first startup takes 30-60 seconds for database migrations

## Step 4: Create Your Admin Account

```bash
# Access the Plausible container
docker exec -it plausible /bin/sh

# Create an admin user
/entrypoint.sh rpc "Plausible.Auth.create_user(\"admin@yourdomain.com\", \"yourpassword\", [role: :admin])"
exit
```

Or visit `http://your-host:8000` - Plausible will prompt you to create the first user on the registration page.

## Step 5: Add Your Website

1. Log in to `https://plausible.yourdomain.com`
2. Click **Add a website**
3. Enter your domain: `yourwebsite.com`
4. Plausible provides a tracking script

## Step 6: Add the Tracking Script

Add this snippet to your website's `<head>`:

```html
<!-- Plausible Analytics tracking script -->
<script defer
  data-domain="yourwebsite.com"
  src="https://plausible.yourdomain.com/js/script.js">
</script>
```

For single-page applications:

```html
<!-- SPA tracking with hash-based routing -->
<script defer
  data-domain="yourwebsite.com"
  src="https://plausible.yourdomain.com/js/script.hash.js">
</script>
```

## Step 7: Configure Reverse Proxy (Nginx)

```nginx
server {
    listen 443 ssl;
    server_name plausible.yourdomain.com;

    ssl_certificate /etc/ssl/certs/plausible.crt;
    ssl_certificate_key /etc/ssl/private/plausible.key;

    location / {
        proxy_pass http://localhost:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## Step 8: Disable New Registrations

After setting up your account, prevent others from registering:

In Portainer, edit the stack and add:

```yaml
environment:
  DISABLE_REGISTRATION: "invite_only"  # or "true" to disable completely
```

## Conclusion

Plausible Analytics running via Portainer gives you a privacy-respecting, cookie-free analytics platform that you fully control. With ClickHouse as the event store, it scales to billions of events while keeping resource usage minimal - and Portainer makes ongoing management, updates, and log inspection straightforward.
