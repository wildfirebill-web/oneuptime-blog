# How to Deploy Seafile via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Seafile, Cloud Storage, Self-Hosted, File Sync

Description: Deploy Seafile via Portainer as a high-performance self-hosted file sync and share solution with end-to-end encryption support and desktop client integration.

## Introduction

Seafile is a high-performance, reliable file sync and share platform. It offers end-to-end encryption for private libraries, fast delta sync, and native clients for all platforms. Deploying via Portainer with MySQL gives you a production-ready file storage solution.

## Deploy as a Stack

```yaml
version: "3.8"

services:
  seafile:
    image: seafileltd/seafile-mc:10.0-latest
    container_name: seafile
    environment:
      DB_HOST: seafile-db
      DB_ROOT_PASSWD: db_root_password
      SEAFILE_ADMIN_EMAIL: admin@example.com
      SEAFILE_ADMIN_PASSWORD: admin_password
      SEAFILE_SERVER_HOSTNAME: seafile.example.com
      SEAFILE_SERVER_LETSENCRYPT: "false"  # Handle SSL externally
      TIME_ZONE: America/New_York
    volumes:
      - seafile_data:/shared
    ports:
      - "80:80"
    depends_on:
      - seafile-db
      - memcached
    restart: unless-stopped

  seafile-db:
    image: mariadb:10.11
    container_name: seafile-db
    environment:
      MYSQL_ROOT_PASSWORD: db_root_password
      MYSQL_LOG_CONSOLE: true
    volumes:
      - seafile_db:/var/lib/mysql
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "--silent"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Memcached for session caching
  memcached:
    image: memcached:1.6-alpine
    container_name: seafile-memcached
    command: memcached -m 256
    restart: unless-stopped

volumes:
  seafile_data:
  seafile_db:
```

## Initial Configuration

After deployment, access Seafile at `http://<host>:80`.

Log in with your admin credentials and configure:

1. **System Settings**: Set the service URL
2. **Libraries**: Create your first encrypted library
3. **Users**: Invite team members

## Seafile CLI Configuration

```bash
# Access Seafile container
docker exec -it seafile bash

# Check service status
/opt/seafile/seafile-server-latest/seafile.sh status
/opt/seafile/seafile-server-latest/seahub.sh status

# Create a user
/opt/seafile/seafile-server-latest/reset-admin.sh
```

## Enabling HTTPS with Nginx Proxy

```yaml
services:
  nginx:
    image: nginx:alpine
    volumes:
      - ./seafile-nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - ./certs:/etc/nginx/certs:ro
    ports:
      - "443:443"
    depends_on:
      - seafile
```

Nginx configuration (`seafile-nginx.conf`):

```nginx
server {
    listen 443 ssl;
    server_name seafile.example.com;

    ssl_certificate /etc/nginx/certs/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/privkey.pem;

    proxy_set_header X-Forwarded-For $remote_addr;

    location / {
        proxy_pass http://seafile:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        client_max_body_size 10G;
        proxy_read_timeout 3600;
    }
}
```

## Desktop Client Setup

Download Seafile desktop client from seafile.com:

1. Install on Windows, Mac, or Linux
2. Add server: `https://seafile.example.com`
3. Log in with your account
4. Select libraries to sync

## Backup Strategy

```bash
# Stop Seafile before backup
docker stop seafile

# Backup data directory
tar czf /backups/seafile-data-$(date +%Y%m%d).tar.gz \
  /var/lib/docker/volumes/seafile_data/_data/seafile-data

# Backup database
docker exec seafile-db mysqldump \
  -u root -pdb_root_password \
  --all-databases > /backups/seafile-db-$(date +%Y%m%d).sql

# Restart Seafile
docker start seafile
```

## Conclusion

Seafile deployed via Portainer provides a fast, reliable file sync platform that excels at handling large files and many small files alike. Its delta-sync algorithm is more efficient than rsync for large binary files, making it ideal for design assets, video files, and software repositories. End-to-end encryption for selected libraries provides strong privacy guarantees for sensitive data.
