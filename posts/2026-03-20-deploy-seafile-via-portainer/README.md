# How to Deploy Seafile via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Seafile, Self-Hosting, Docker, File Sync, Docker Compose

Description: Learn how to deploy Seafile, the high-performance file synchronization and sharing platform, using Portainer's stack deployment with MariaDB and Memcached.

---

Seafile offers fast file sync with built-in encryption and team collaboration features. Unlike Nextcloud it focuses purely on file storage, giving it better performance for large datasets. This guide deploys the community edition via Portainer.

## Prerequisites

- Portainer running on a host with at least 2GB RAM
- Port 80 or 443 available
- A domain name for the `SERVICE_URL` variable

## Compose Stack

In Portainer go to **Stacks > Add Stack**. Paste the following YAML. Seafile requires both a relational database and a Memcached instance for session caching:

```yaml
version: "3.8"

services:
  db:
    image: mariadb:10.11
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: db_dev@123      # Change this
      MYSQL_LOG_CONSOLE: "true"
    volumes:
      - seafile_db:/var/lib/mysql

  memcached:
    image: memcached:1.6-alpine
    restart: always
    # Allocate 256MB to memcached for session caching
    command: memcached -m 256

  seafile:
    image: seafileltd/seafile-mc:latest
    restart: always
    ports:
      - "8081:80"
    depends_on:
      - db
      - memcached
    volumes:
      - seafile_data:/shared
    environment:
      DB_HOST: db
      DB_ROOT_PASSWD: db_dev@123           # Must match db service
      TIME_ZONE: UTC
      SEAFILE_ADMIN_EMAIL: admin@example.com
      SEAFILE_ADMIN_PASSWORD: adminpass    # Change this
      SEAFILE_SERVER_LETSENCRYPT: "false"
      SERVICE_URL: http://seafile.example.com:8081

volumes:
  seafile_db:
  seafile_data:
```

## Deploying

1. In Portainer go to **Stacks > Add Stack**.
2. Name the stack `seafile` and paste the YAML above.
3. Update `SERVICE_URL` to your actual server address.
4. Click **Deploy the stack**.

Initialization takes about 60 seconds. Watch progress in **Containers > seafile > Logs**.

## Configuring HTTPS

For production, place Seafile behind an Nginx or Traefik reverse proxy and set:

```yaml
SERVICE_URL: https://seafile.example.com
SEAFILE_SERVER_LETSENCRYPT: "true"
```

## Monitoring

Use OneUptime to create an HTTP monitor on `http://<host>:8081`. A 200 response on the login page confirms Seafile is operational. Add an alert for response times above 5 seconds to catch database saturation early.
