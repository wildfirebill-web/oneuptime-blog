# How to Deploy Ghost Blog with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Ghost, Docker, Blog, Self-Hosted, Docker Compose

Description: Learn how to deploy the Ghost blogging platform with a MySQL database using Portainer stacks for a self-hosted blog.

---

Ghost is a modern, open-source publishing platform built on Node.js. Deploying it with Portainer stacks gives you a self-hosted blog with full control over your data, plugins, and customization.

---

## Deploy Ghost with MySQL via Portainer Stack

In Portainer: **Stacks** → **Add stack** → paste:

```yaml
version: "3.8"

services:
  ghost:
    image: ghost:5-alpine
    container_name: ghost
    restart: unless-stopped
    ports:
      - "2368:2368"
    environment:
      database__client: mysql
      database__connection__host: db
      database__connection__user: ghost
      database__connection__password: ${GHOST_DB_PASSWORD}
      database__connection__database: ghost
      url: https://blog.example.com
      mail__transport: SMTP
      mail__options__host: ${SMTP_HOST}
      mail__options__port: "587"
      mail__options__auth__user: ${SMTP_USER}
      mail__options__auth__pass: ${SMTP_PASS}
    volumes:
      - ghost-content:/var/lib/ghost/content
    depends_on:
      - db

  db:
    image: mysql:8.0
    container_name: ghost-db
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ghost
      MYSQL_USER: ghost
      MYSQL_PASSWORD: ${GHOST_DB_PASSWORD}
    volumes:
      - ghost-db:/var/lib/mysql

volumes:
  ghost-content:
  ghost-db:
```

Set environment variables in Portainer's stack environment editor.

---

## Add Traefik Labels for HTTPS

```yaml
services:
  ghost:
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.ghost.rule=Host(`blog.example.com`)"
      - "traefik.http.routers.ghost.tls.certresolver=letsencrypt"
      - "traefik.http.services.ghost.loadbalancer.server.port=2368"
```

---

## Access and Configure Ghost

1. Navigate to `https://blog.example.com/ghost` for the admin panel.
2. Complete the setup wizard to create your first admin account.
3. Install themes from the Ghost Marketplace or upload custom ones.

---

## Backup Ghost Data

```bash
# Backup the content volume
docker run --rm   -v ghost-content:/data   -v $(pwd):/backup   alpine tar czf /backup/ghost-backup-$(date +%Y%m%d).tar.gz -C /data .

# Backup MySQL
docker exec ghost-db mysqldump -u ghost -p${GHOST_DB_PASSWORD} ghost > ghost-db-backup.sql
```

---

## Summary

Deploy Ghost with the `ghost:5-alpine` image and MySQL 8 using a Portainer stack with named volumes for data persistence. Set `url` to your public domain for correct link generation. Add Traefik labels for automatic HTTPS. Backup both the `ghost-content` volume and the MySQL database regularly.
