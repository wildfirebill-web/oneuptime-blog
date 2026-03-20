# How to Deploy WordPress via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, WordPress, CMS, Self-Hosted, Web

Description: Deploy WordPress with MySQL and Redis cache via Portainer for a complete, performant self-hosted blog and content management system.

## Introduction

WordPress powers over 40% of the web. Deploying it via Portainer with MySQL and Redis gives you a complete, production-ready CMS stack with object caching. Persistent volumes ensure your content survives container updates.

## Deploy as a Stack

```yaml
version: "3.8"

services:
  wordpress:
    image: wordpress:6.5-php8.3-apache
    container_name: wordpress
    environment:
      WORDPRESS_DB_HOST: wordpress-db
      WORDPRESS_DB_NAME: wordpress
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: wp_password
      WORDPRESS_DEBUG: 0
      # Redis cache settings
      WORDPRESS_CONFIG_EXTRA: |
        define('WP_REDIS_HOST', 'redis');
        define('WP_REDIS_PORT', 6379);
        define('WP_REDIS_PASSWORD', 'redis_password');
        define('WP_CACHE', true);
        define('WP_MEMORY_LIMIT', '256M');
        define('DISALLOW_FILE_EDIT', true);
    volumes:
      - wordpress_data:/var/www/html
      - ./uploads.ini:/usr/local/etc/php/conf.d/uploads.ini:ro
    ports:
      - "8080:80"
    depends_on:
      - wordpress-db
      - redis
    restart: unless-stopped

  wordpress-db:
    image: mysql:8.4
    container_name: wordpress-db
    environment:
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: wp_password
      MYSQL_ROOT_PASSWORD: root_password
    volumes:
      - wordpress_db:/var/lib/mysql
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "wordpress", "-pwp_password"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: redis
    command: redis-server --requirepass redis_password
    restart: unless-stopped

volumes:
  wordpress_data:
  wordpress_db:
```

## PHP Configuration

Create `uploads.ini`:

```ini
; PHP configuration for WordPress
upload_max_filesize = 64M
post_max_size = 64M
max_execution_time = 300
memory_limit = 256M
max_input_vars = 5000
```

## Post-Deployment Setup

1. Access WordPress at `http://<host>:8080`
2. Complete the WordPress installation wizard
3. Install the **Redis Object Cache** plugin:
   - Go to **Plugins > Add New**
   - Search for "Redis Object Cache"
   - Install and activate
   - Enable the cache in the plugin settings

## WordPress Security Hardening

Add to your `WORDPRESS_CONFIG_EXTRA`:

```php
// Limit login attempts
define('WP_AUTO_UPDATE_CORE', true);

// Disable XML-RPC
add_filter('xmlrpc_enabled', '__return_false');

// Hide WordPress version
remove_action('wp_head', 'wp_generator');

// Enforce HTTPS for admin
define('FORCE_SSL_ADMIN', true);
```

## Adding Behind Traefik

```yaml
services:
  wordpress:
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.wordpress.rule=Host(`blog.example.com`)"
      - "traefik.http.routers.wordpress.entrypoints=websecure"
      - "traefik.http.routers.wordpress.tls.certresolver=letsencrypt"
      - "traefik.http.services.wordpress.loadbalancer.server.port=80"
    networks:
      - traefik-public
      - default
```

## Backups

```bash
# Backup WordPress files
docker run --rm \
  -v wordpress_data:/source \
  -v /backups:/backup \
  alpine tar czf /backup/wordpress-$(date +%Y%m%d).tar.gz /source

# Backup WordPress database
docker exec wordpress-db mysqldump \
  -u wordpress -pwp_password wordpress \
  > /backups/wordpress-db-$(date +%Y%m%d).sql
```

## Conclusion

WordPress deployed via Portainer with MySQL and Redis provides a complete, performant blogging and CMS platform. Redis object caching significantly improves page load times for dynamic content. The persistent volumes ensure your posts, themes, and plugins survive container updates, and the clean Portainer stack definition makes migration to a new host straightforward.
