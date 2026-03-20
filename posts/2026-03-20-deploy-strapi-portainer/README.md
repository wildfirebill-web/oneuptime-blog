# How to Deploy Strapi CMS via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Strapi, CMS, Docker, Headless CMS

Description: Deploy Strapi open-source headless CMS using Portainer backed by PostgreSQL for content management.

## Introduction

Strapi is the leading open-source headless CMS built with Node.js. It generates REST and GraphQL APIs for your content types and provides an admin panel for content editors. This guide deploys Strapi backed by PostgreSQL via Portainer.

## Prerequisites

- Portainer installed with Docker
- At least 1 GB RAM

## Step 1: Create the Stack in Portainer

Navigate to **Stacks** > **Add Stack**:

```yaml
# docker-compose.yml - Strapi
version: "3.8"

services:
  strapi:
    image: strapi/strapi:4.22.0-node18-alpine
    container_name: strapi
    restart: unless-stopped
    ports:
      - "1337:1337"
    volumes:
      - strapi_app:/srv/app
    environment:
      - DATABASE_CLIENT=postgres
      - DATABASE_HOST=strapi_postgres
      - DATABASE_PORT=5432
      - DATABASE_NAME=strapi
      - DATABASE_USERNAME=strapi
      - DATABASE_PASSWORD=${DB_PASSWORD}
      - DATABASE_SSL=false
      - JWT_SECRET=${JWT_SECRET}
      - ADMIN_JWT_SECRET=${ADMIN_JWT_SECRET}
      - APP_KEYS=${APP_KEYS}
      - API_TOKEN_SALT=${API_TOKEN_SALT}
      - TRANSFER_TOKEN_SALT=${TRANSFER_TOKEN_SALT}
      - NODE_ENV=production
    depends_on:
      strapi_postgres:
        condition: service_healthy
    networks:
      - strapi_net

  strapi_postgres:
    image: postgres:16-alpine
    container_name: strapi_postgres
    restart: unless-stopped
    volumes:
      - strapi_postgres_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=strapi
      - POSTGRES_USER=strapi
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U strapi"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - strapi_net

volumes:
  strapi_app:
  strapi_postgres_data:

networks:
  strapi_net:
    driver: bridge
```

## Step 2: Set Environment Variables in Portainer

Generate secrets using:
```bash
openssl rand -base64 32
```

```
DB_PASSWORD=your-postgres-password
JWT_SECRET=<generated-base64-secret>
ADMIN_JWT_SECRET=<generated-base64-secret>
APP_KEYS=<key1>,<key2>,<key3>,<key4>
API_TOKEN_SALT=<generated-base64-secret>
TRANSFER_TOKEN_SALT=<generated-base64-secret>
```

## Step 3: Access the Strapi Admin Panel

Open `http://<host>:1337/admin` to create your first admin account.

## Step 4: Create a Content Type

1. Go to **Content-Type Builder** in the admin panel
2. Create a **Collection Type** named `Article`
3. Add fields: `title` (Text), `content` (Rich Text), `publishedAt` (Date)
4. Click **Save** — Strapi auto-generates REST and GraphQL endpoints

## Step 5: Use the API

```bash
# Get your API token: Settings > API Tokens > Create new API Token

# List articles (public)
curl http://localhost:1337/api/articles

# List with population of relations
curl http://localhost:1337/api/articles?populate=*

# Create an article (requires token)
curl -X POST http://localhost:1337/api/articles \
  -H 'Authorization: Bearer <api-token>' \
  -H 'Content-Type: application/json' \
  -d '{"data": {"title": "Hello Strapi", "content": "My first article"}}'

# GraphQL query
curl -X POST http://localhost:1337/graphql \
  -H 'Content-Type: application/json' \
  -d '{"query": "{ articles { data { id attributes { title } } } }"}'
```

## Conclusion

Strapi generates REST and GraphQL APIs automatically when you define content types. The `APP_KEYS` env var is used to sign session cookies and should be 4 unique random base64 strings. In production, set `NODE_ENV=production` to disable the content-type builder (to prevent accidental schema changes). Use the `strapi transfer` command to migrate content between environments.
