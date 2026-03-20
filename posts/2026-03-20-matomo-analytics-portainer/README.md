# How to Deploy Matomo Analytics via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Matomo, Analytics, Self-Hosted, Privacy, GDPR

Description: Deploy Matomo, the powerful open-source web analytics platform, as a Docker stack through Portainer for complete ownership of your analytics data.

## Introduction

Matomo (formerly Piwik) is the world's most popular self-hosted web analytics platform. It offers Google Analytics-level features while keeping all data on your own servers. Deploying via Portainer makes it easy to manage, update, and monitor the Matomo stack.

## Prerequisites

- Portainer CE or BE installed
- Docker Engine 20.10+
- A domain name (for HTTPS access)
- SMTP server details (optional, for email reports)

## Step 1: Create Data Directories

```bash
# Create directories for persistent storage
mkdir -p /opt/matomo/data
mkdir -p /opt/matomo/db
```

## Step 2: Create the Stack in Portainer

Navigate to **Stacks** → **Add Stack** → **Web Editor**:

```yaml
version: "3.8"

services:
  # MariaDB — Matomo's database
  matomo-db:
    image: mariadb:10.11
    container_name: matomo-db
    restart: unless-stopped
    command: --max-allowed-packet=64MB
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: matomo
      MYSQL_USER: matomo
      MYSQL_PASSWORD: matomopassword
    volumes:
      - /opt/matomo/db:/var/lib/mysql
    networks:
      - matomo-net
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Matomo Analytics application
  matomo:
    image: matomo:5.1
    container_name: matomo
    restart: unless-stopped
    depends_on:
      matomo-db:
        condition: service_healthy
    ports:
      - "8080:80"
    environment:
      # Database connection settings
      MATOMO_DATABASE_HOST: matomo-db
      MATOMO_DATABASE_PORT: "3306"
      MATOMO_DATABASE_USERNAME: matomo
      MATOMO_DATABASE_PASSWORD: matomopassword
      MATOMO_DATABASE_DBNAME: matomo

      # PHP settings for large traffic sites
      PHP_MEMORY_LIMIT: "512M"
      PHP_MAX_EXECUTION_TIME: "300"
    volumes:
      - /opt/matomo/data:/var/www/html
    networks:
      - matomo-net

  # Optional: Nginx reverse proxy for HTTPS
  matomo-nginx:
    image: nginx:alpine
    container_name: matomo-nginx
    restart: unless-stopped
    depends_on:
      - matomo
    ports:
      - "443:443"
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - /etc/ssl/certs:/etc/ssl/certs:ro
    networks:
      - matomo-net

networks:
  matomo-net:
    driver: bridge
```

## Step 3: Deploy the Stack

1. Name the stack `matomo`
2. Click **Deploy the stack**
3. Wait for the database to initialize (~30 seconds)

## Step 4: Complete the Web Installer

1. Open `http://your-host:8080`
2. Follow the installation wizard:
   - **Database setup**: Use the credentials from the compose file
   - **Create admin account**: Set a strong password
   - **Set up first website**: Enter your website's URL and name
3. Matomo generates a tracking code — save it

## Step 5: Add the Tracking Code

```html
<!-- Matomo tracking code — add before </head> -->
<script>
  var _paq = window._paq = window._paq || [];
  _paq.push(['trackPageView']);
  _paq.push(['enableLinkTracking']);
  (function() {
    var u="https://matomo.yourdomain.com/";
    _paq.push(['setTrackerUrl', u+'matomo.php']);
    _paq.push(['setSiteId', '1']);
    var d=document, g=d.createElement('script'), s=d.getElementsByTagName('script')[0];
    g.async=true; g.src=u+'matomo.js'; s.parentNode.insertBefore(g,s);
  })();
</script>
```

## Step 6: Configure Archiving (Cron Job)

For better performance, disable browser-triggered archiving and use a cron job:

```bash
# On the host, add to crontab
crontab -e

# Run Matomo archiving every 5 minutes
*/5 * * * * docker exec matomo php /var/www/html/console core:archive --url=https://matomo.yourdomain.com > /dev/null 2>&1
```

In Matomo admin: **System** → **General Settings** → disable **Browser Trigger Archiving**.

## Step 7: Enable GeoIP Location

```bash
# Inside the Matomo container, download GeoIP database
docker exec -it matomo bash
# Then download and configure in Admin > Geolocation
```

Or configure via the admin panel:
1. **System** → **Geolocation**
2. Enable **MaxMind GeoIP2** or **DB-IP**
3. Follow the provider's setup instructions

## Step 8: Set Up Email Reports

1. **Personal** → **Email Reports** → **Add New Report**
2. Configure weekly or monthly analytics digests
3. Requires SMTP configured under **System** → **General Settings** → **Email Server Settings**

## Step 9: Import Historical Google Analytics Data

Matomo provides an official Google Analytics importer plugin:

1. **Marketplace** → search "Google Analytics Importer"
2. Install and activate
3. Follow the OAuth setup to grant Matomo access to your GA data
4. Start the import job

## Conclusion

Matomo deployed via Portainer gives you a full-featured, self-hosted analytics platform with complete data ownership. Unlike SaaS analytics tools, Matomo stores everything in your own MariaDB database and Portainer makes it easy to monitor resource usage, view logs, and update the stack when new versions are released.
