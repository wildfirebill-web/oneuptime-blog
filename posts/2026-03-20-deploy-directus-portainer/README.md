# How to Deploy Directus via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Directus, CMS, Docker, Headless CMS

Description: Deploy Directus data platform and headless CMS using Portainer for instant API generation over your database.

## Introduction

Deploy Directus data platform and headless CMS using Portainer for instant API generation over your database. Portainer's stack management capabilities make deploying and managing Directus straightforward for development and production environments.

## Prerequisites

- Portainer installed with Docker
- At least 2-4 GB RAM
- Sufficient disk space for data storage

## Step 1: Deploy via Portainer Stack

Navigate to **Stacks** > **Add Stack** in Portainer:

```yaml
# docker-compose.yml
version: "3.8"

services:
  app:
    image: app-image:latest
    container_name: directus
    restart: always
    ports:
      - "7700:7700"
    volumes:
      - app-data:/data
    environment:
      - MASTER_KEY=${MASTER_KEY}
      - ENV=production
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:7700/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    logging:
      driver: json-file
      options:
        max-size: "100m"
        max-file: "3"
    networks:
      - app-net

  # PostgreSQL for relational data (if needed)
  postgres:
    image: postgres:15-alpine
    restart: always
    environment:
      - POSTGRES_DB=appdb
      - POSTGRES_USER=appuser
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - app-net

volumes:
  app-data:
  postgres-data:

networks:
  app-net:
    driver: bridge
```

## Step 2: Configure Environment Variables

Set the required environment variables in Portainer's stack editor:

```bash
MASTER_KEY=your-secure-master-key
DB_PASSWORD=secure-database-password
APP_URL=https://app.example.com
```

## Step 3: Initialize the Application

After deployment, complete initial setup:

```bash
# Access the container via Portainer console
# Portainer > Containers > app > Console

# Check application health
curl http://localhost:7700/health

# View startup logs
docker logs directus --tail 50
```

## Step 4: Configure and Test

Test the application functionality:

```bash
# Create a test index/collection
curl -X POST 'http://localhost:7700/indexes'   -H 'Authorization: Bearer your-master-key'   -H 'Content-Type: application/json'   --data-binary '{"uid": "test", "primaryKey": "id"}'

# Add test documents
curl -X POST 'http://localhost:7700/indexes/test/documents'   -H 'Authorization: Bearer your-master-key'   -H 'Content-Type: application/json'   --data-binary '[{"id": 1, "name": "test document", "content": "hello world"}]'

# Search
curl 'http://localhost:7700/indexes/test/search?q=hello'   -H 'Authorization: Bearer your-master-key'
```

## Step 5: Set Up Reverse Proxy

Expose the service securely via Nginx or Traefik:

```yaml
# Add to your stack
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - app
    networks:
      - app-net
```

```nginx
# nginx.conf
server {
    listen 443 ssl;
    server_name app.example.com;
    
    ssl_certificate /etc/nginx/certs/cert.pem;
    ssl_certificate_key /etc/nginx/certs/key.pem;
    
    location / {
        proxy_pass http://app:7700;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## Step 6: Configure Backups

Automate data backups:

```bash
#!/bin/bash
# backup.sh
BACKUP_DIR="/backups/directus"
DATE=$(date +%Y%m%d_%H%M%S)
mkdir -p $BACKUP_DIR

docker run --rm   -v app-data:/source:ro   -v $BACKUP_DIR:/backup   alpine tar czf /backup/data-$DATE.tar.gz -C /source .

echo "Backup complete: $BACKUP_DIR/data-$DATE.tar.gz"

# Retain 7 days
find $BACKUP_DIR -name "*.tar.gz" -mtime +7 -delete
```

## Step 7: Update the Application

Update to newer versions via Portainer:

1. Edit the stack in Portainer
2. Update the image tag to the new version
3. Click **Update the stack**
4. Monitor logs during the update

## Conclusion

Deploying Directus via Portainer provides a well-managed, production-ready instance that's easy to maintain. The persistent volume configuration ensures data survives container restarts and updates, while Portainer's visual interface simplifies ongoing management tasks. With proper backup automation and a reverse proxy for SSL termination, this deployment is suitable for production use.
