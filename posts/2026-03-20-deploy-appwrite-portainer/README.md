# How to Deploy Appwrite via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Appwrite, Backend-as-a-Service, Docker, Self-Hosted

Description: Deploy Appwrite open-source backend-as-a-service platform using Portainer for authentication, databases, storage, and serverless functions.

## Introduction

Appwrite is a self-hosted backend-as-a-service (BaaS) that provides authentication, databases, file storage, serverless functions, and real-time APIs. This guide deploys Appwrite using its official Docker Compose configuration via Portainer.

## Prerequisites

- Portainer installed with Docker
- Minimum 2 CPUs and 4 GB RAM
- A domain name (optional, for TLS)

## Step 1: Create the Stack in Portainer

Navigate to **Stacks** > **Add Stack**:

```yaml
# docker-compose.yml - Appwrite

version: "3.8"

services:
  appwrite:
    image: appwrite/appwrite:1.5.7
    container_name: appwrite
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - appwrite_uploads:/storage/uploads
      - appwrite_cache:/storage/cache
      - appwrite_config:/storage/config
      - appwrite_certificates:/storage/certificates
    environment:
      - _APP_ENV=production
      - _APP_OPENSSL_KEY_V1=${APPWRITE_OPENSSL_KEY}
      - _APP_DOMAIN=${APPWRITE_DOMAIN}
      - _APP_DOMAIN_TARGET=${APPWRITE_DOMAIN}
      - _APP_REDIS_HOST=appwrite-redis
      - _APP_REDIS_PORT=6379
      - _APP_DB_HOST=appwrite-mariadb
      - _APP_DB_PORT=3306
      - _APP_DB_USER=appwrite
      - _APP_DB_PASS=${DB_PASSWORD}
      - _APP_DB_SCHEMA=appwrite
    depends_on:
      - appwrite-mariadb
      - appwrite-redis
    networks:
      - appwrite_net

  appwrite-mariadb:
    image: mariadb:10.11
    container_name: appwrite_mariadb
    restart: unless-stopped
    volumes:
      - appwrite_mariadb:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - MYSQL_DATABASE=appwrite
      - MYSQL_USER=appwrite
      - MYSQL_PASSWORD=${DB_PASSWORD}
    healthcheck:
      test: ["CMD", "healthcheck.sh", "--connect", "--innodb_initialized"]
      interval: 30s
      timeout: 10s
      retries: 5
    networks:
      - appwrite_net

  appwrite-redis:
    image: redis:7-alpine
    container_name: appwrite_redis
    restart: unless-stopped
    command: redis-server --maxmemory 512mb --maxmemory-policy allkeys-lru
    volumes:
      - appwrite_redis:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - appwrite_net

  appwrite-worker-functions:
    image: appwrite/appwrite:1.5.7
    container_name: appwrite_worker_functions
    restart: unless-stopped
    entrypoint: worker-functions
    environment:
      - _APP_ENV=production
      - _APP_REDIS_HOST=appwrite-redis
      - _APP_REDIS_PORT=6379
      - _APP_DB_HOST=appwrite-mariadb
      - _APP_DB_PORT=3306
      - _APP_DB_USER=appwrite
      - _APP_DB_PASS=${DB_PASSWORD}
      - _APP_DB_SCHEMA=appwrite
    depends_on:
      - appwrite-mariadb
      - appwrite-redis
    networks:
      - appwrite_net

volumes:
  appwrite_uploads:
  appwrite_cache:
  appwrite_config:
  appwrite_certificates:
  appwrite_mariadb:
  appwrite_redis:

networks:
  appwrite_net:
    driver: bridge
```

## Step 2: Set Environment Variables in Portainer

```text
APPWRITE_OPENSSL_KEY=your-random-32-char-secret
APPWRITE_DOMAIN=appwrite.yourdomain.com
DB_PASSWORD=your-db-password
MYSQL_ROOT_PASSWORD=your-root-password
```

## Step 3: Access the Console

Open `http://<host>` in your browser. Complete the first-run setup to create an admin account.

## Step 4: Create a Project via CLI

```bash
# Install Appwrite CLI
npm install -g appwrite-cli

# Login
appwrite login

# Initialize a project
appwrite init project
```

## Step 5: Test the API

```bash
# Create a document via REST API
curl -X POST 'https://appwrite.yourdomain.com/v1/databases/{databaseId}/collections/{collectionId}/documents' \
  -H 'X-Appwrite-Project: <project-id>' \
  -H 'X-Appwrite-Key: <api-key>' \
  -H 'Content-Type: application/json' \
  -d '{"documentId":"unique()", "data": {"name": "Alice"}}'
```

## Conclusion

Appwrite provides a complete backend stack (database, auth, storage, functions) as a single self-hosted deployment. The worker services handle asynchronous tasks like function execution and email sending. For production, configure `_APP_SMTP_HOST` for email delivery and `_APP_STORAGE_LIMIT` to control file upload sizes.
