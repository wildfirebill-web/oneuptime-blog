# How to Deploy MinIO for ML Model Storage via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, MinIO, Machine Learning, S3, Docker

Description: Deploy MinIO as S3-compatible object storage for machine learning model artifacts and datasets using Portainer.

## Introduction

Deploy MinIO as S3-compatible object storage for machine learning model artifacts and datasets using Portainer. This guide provides comprehensive instructions for deploying and integrating this tool into your workflow.

## Prerequisites

- Portainer installed with Docker
- At least 4 GB RAM
- Sufficient disk space for data storage
- Basic understanding of Docker and containerization

## Step 1: Deploy via Portainer Stack

Create a new stack in Portainer with this docker-compose.yml:

```yaml
version: "3.8"

services:
  app:
    image: appropriate-image:latest
    container_name: ml-tool
    restart: always
    ports:
      - "8080:8080"
    volumes:
      - app-data:/data
    environment:
      - SECRET_KEY=${SECRET_KEY}
      - DATABASE_URL=postgresql://user:pass@postgres:5432/db
    depends_on:
      - postgres
    networks:
      - ml-net

  postgres:
    image: postgres:15-alpine
    container_name: ml-postgres
    restart: always
    volumes:
      - postgres-data:/var/lib/postgresql/data
    environment:
      - POSTGRES_USER=mluser
      - POSTGRES_PASSWORD=${DB_PASSWORD}
      - POSTGRES_DB=mldb
    networks:
      - ml-net

  redis:
    image: redis:7-alpine
    container_name: ml-redis
    restart: always
    volumes:
      - redis-data:/data
    networks:
      - ml-net

volumes:
  app-data:
  postgres-data:
  redis-data:

networks:
  ml-net:
    driver: bridge
```

## Step 2: Configure Environment Variables

In Portainer's stack editor, configure:

```bash
SECRET_KEY=your-secret-key-here
DB_PASSWORD=secure-database-password
```

## Step 3: Initialize the Application

After deployment, run initialization commands:

```bash
# Access container via Portainer console
# Run database migrations
docker exec ml-tool python manage.py migrate

# Create admin user
docker exec -it ml-tool python manage.py createsuperuser

# Verify installation
curl http://localhost:8080/health
```

## Step 4: Configure Storage

Set up persistent storage for models and data:

```yaml
# Configure volume with specific host path
volumes:
  app-data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /data/ml-storage
```

```bash
# Create and set permissions
sudo mkdir -p /data/ml-storage
sudo chown 1000:1000 /data/ml-storage
```

## Step 5: Set Up Authentication

Configure user authentication and access control:

```yaml
# Environment variables for auth configuration
services:
  app:
    environment:
      - AUTH_ENABLED=true
      - ADMIN_USERNAME=admin
      - ADMIN_EMAIL=admin@example.com
```

## Step 6: Configure Backups

Set up automated backups via Portainer:

```bash
#!/bin/bash
# backup.sh
BACKUP_DIR="/backups/ml-tool"
DATE=$(date +%Y%m%d_%H%M%S)
mkdir -p $BACKUP_DIR

# Backup database
docker exec ml-postgres pg_dump -U mluser mldb |   gzip > $BACKUP_DIR/db-$DATE.sql.gz

# Backup application data
docker run --rm   -v app-data:/source:ro   -v $BACKUP_DIR:/backup   alpine tar czf /backup/data-$DATE.tar.gz -C /source .

echo "Backup complete"
```

## Step 7: Monitor Performance

Track application metrics through Portainer:

1. Go to **Containers** > select your container
2. Click **Stats** to view real-time resource usage
3. Set up Prometheus monitoring for detailed metrics

## Step 8: Update and Maintenance

Update to new versions via Portainer:

1. Edit the stack in Portainer
2. Update image tags to new version
3. Click **Update the stack**
4. Monitor container logs during update

## Conclusion

Deploying this ML tool via Portainer provides a production-ready, manageable service that integrates into your existing container infrastructure. Portainer's stack management simplifies updates, configuration management, and troubleshooting while providing a unified interface for your entire ML platform.
