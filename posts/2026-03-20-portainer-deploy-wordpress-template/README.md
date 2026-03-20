# How to Deploy WordPress Using a Portainer Template

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, WordPress, Templates, DevOps

Description: Step-by-step guide to deploying a production-ready WordPress site using Portainer's stack template feature.

## Introduction

WordPress is one of the most popular applications deployed in Docker environments. Portainer's template system makes it easy to deploy a complete WordPress stack - including the database - in minutes. This guide walks through deploying WordPress using a Portainer template and configuring it for production use.

## Prerequisites

- Portainer CE or BE installed and running
- Docker standalone environment connected
- A domain name (optional, but recommended for production)
- At least 1GB RAM available on the host

## Step 1: Access App Templates

1. In Portainer, select your Docker environment
2. Click **App Templates** in the sidebar
3. Search for "WordPress"
4. Select the **WordPress** stack template

## Step 2: Configure the WordPress Stack

Fill in the template variables:

```text
Stack name:           wordpress-site

WordPress port:       80          (or 8080 to avoid conflicts)
WordPress DB host:    db
WordPress DB name:    wordpress
WordPress DB user:    wpuser

MySQL root password:  [generate-strong-password]
MySQL database:       wordpress
MySQL user:           wpuser
MySQL password:       [generate-strong-password]
```

**Tip:** Use a password manager to generate strong passwords (at least 24 characters with mixed case, numbers, and symbols).

## Step 3: Deploy the Stack

Click **Deploy the stack**. The deployment process:

1. Creates the `wordpress` and `db` networks
2. Creates named volumes for data persistence
3. Pulls `wordpress:latest` and `mysql:8` images
4. Starts both containers

Watch the output for any errors.

## Step 4: Complete WordPress Installation

1. Open your browser and navigate to `http://your-host:80`
2. The WordPress installation wizard appears
3. Complete the setup:

```text
Site Title:      My WordPress Site
Username:        admin
Password:        [strong-admin-password]
Email:           admin@example.com
```

4. Click **Install WordPress**

## Step 5: Configure WordPress for Production

### Set the Correct URL

After installation, update the site URL:

```php
// In wp-admin → Settings → General:
WordPress Address (URL): https://myblog.example.com
Site Address (URL):      https://myblog.example.com
```

### Install Essential Plugins

Via `wp-admin → Plugins → Add New`, install:

- **Wordfence Security** - Firewall and malware scanner
- **UpdraftPlus** - Backup solution
- **WP Super Cache** - Caching for performance
- **Yoast SEO** - SEO optimization

## Step 6: Set Up a Reverse Proxy with SSL

For production, place WordPress behind Nginx Proxy Manager or Traefik:

```yaml
# Add to your WordPress stack Compose file

services:
  wordpress:
    image: wordpress:latest
    # Remove direct port exposure - let the proxy handle it
    expose:
      - "80"
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_NAME: wordpress
      WORDPRESS_DB_USER: wpuser
      WORDPRESS_DB_PASSWORD: ${MYSQL_PASSWORD}
      # Required for HTTPS behind a proxy
      WORDPRESS_CONFIG_EXTRA: |
        define('FORCE_SSL_ADMIN', true);
        if (strpos($_SERVER['HTTP_X_FORWARDED_PROTO'], 'https') !== false) {
          $_SERVER['HTTPS'] = 'on';
        }
    labels:
      # Traefik labels for automatic HTTPS
      - "traefik.enable=true"
      - "traefik.http.routers.wordpress.rule=Host(`myblog.example.com`)"
      - "traefik.http.routers.wordpress.tls.certresolver=letsencrypt"
    networks:
      - wordpress-net
      - traefik-net   # Connect to Traefik's network

  db:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wpuser
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
    volumes:
      - mysql-data:/var/lib/mysql
    networks:
      - wordpress-net

volumes:
  mysql-data:
  wordpress-data:

networks:
  wordpress-net:
  traefik-net:
    external: true    # Use existing Traefik network
```

## Step 7: Configure Persistent Storage

Ensure WordPress files are persisted for themes, plugins, and uploads:

```yaml
services:
  wordpress:
    volumes:
      # Persist entire WordPress installation
      - wordpress-data:/var/www/html
      # Or just persist uploads
      - wp-uploads:/var/www/html/wp-content/uploads

volumes:
  wordpress-data:
    driver: local
  wp-uploads:
    driver: local
```

## Step 8: Set Up Automatic Backups

```yaml
services:
  # Add a backup service to your stack
  backup:
    image: alpine:latest
    volumes:
      - wordpress-data:/wordpress:ro
      - mysql-data:/mysql:ro
      - /data/backups:/backups
    command: >
      sh -c "tar czf /backups/wordpress-$$(date +%Y%m%d).tar.gz /wordpress &&
             echo 'Backup complete'"
    # Run as a one-shot or scheduled task
```

For automated daily backups, use a separate Docker container with a cron schedule:

```bash
# Manual backup via Portainer console or CLI
docker exec wordpress-site_db_1 \
  mysqldump -u wpuser -p${MYSQL_PASSWORD} wordpress > \
  /data/backups/wordpress-$(date +%Y%m%d).sql
```

## Step 9: Update WordPress

To update WordPress:

1. In the Portainer stack editor, update the image tag:

```yaml
services:
  wordpress:
    image: wordpress:6.4  # Specify a version or use :latest
```

2. Click **Update the stack**
3. Portainer pulls the new image and recreates the container
4. WordPress auto-updates its database if needed

## Monitoring Your WordPress Stack

Use Portainer's container stats to monitor resource usage:

1. Go to **Stacks → wordpress-site**
2. Click on the `wordpress` container
3. Click **Stats** to see CPU, memory, and network usage

For production, set memory limits:

```yaml
services:
  wordpress:
    deploy:
      resources:
        limits:
          memory: 512m
        reservations:
          memory: 256m
```

## Conclusion

Deploying WordPress with Portainer's template system is straightforward and takes only minutes. The template handles the multi-service setup automatically, leaving you to focus on WordPress configuration and content. For production deployments, add SSL via a reverse proxy, configure persistent volumes, set up automated backups, and monitor resource usage through Portainer's stats interface.
