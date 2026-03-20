# How to Deploy Multi-Service Applications as Stacks in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Docker Compose, Stacks, DevOps

Description: Learn how to deploy multi-service applications as stacks in Portainer using Docker Compose definitions.

## Introduction

Portainer Stacks provide a powerful way to manage multi-service applications using Docker Compose syntax. Instead of managing each container individually, stacks let you define, deploy, and manage entire application environments as a single unit. This guide walks you through deploying a multi-service application as a stack in Portainer.

## Prerequisites

- Portainer CE or BE installed and running
- Docker Engine 20.10+ or Docker Swarm cluster
- Basic familiarity with Docker Compose syntax

## Understanding Portainer Stacks

A Portainer Stack is essentially a Docker Compose file managed through the Portainer UI. Stacks can be deployed on:

- **Standalone Docker** - uses Docker Compose v2
- **Docker Swarm** - uses Docker stack deploy under the hood
- **Kubernetes** - converts Compose to Kubernetes manifests (limited support)

## Step 1: Navigate to Stacks

1. Log in to your Portainer instance
2. Select your Docker environment from the Home screen
3. Click **Stacks** in the left sidebar
4. Click **+ Add stack**

## Step 2: Define Your Stack

Choose one of the deployment methods:

- **Web editor** - write the Compose file directly in the browser
- **Upload** - upload a `docker-compose.yml` file
- **Repository** - pull from a Git repository
- **Custom template** - use a saved template

## Step 3: Write Your Docker Compose File

Here is an example multi-service application stack with a web app, API, and database:

```yaml
version: "3.8"

services:
  # Frontend web application
  frontend:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - api
    networks:
      - app-network

  # Backend API service
  api:
    image: node:18-alpine
    working_dir: /app
    command: node server.js
    environment:
      - NODE_ENV=production
      - DB_HOST=database
      - DB_PORT=5432
      - DB_NAME=appdb
    depends_on:
      - database
    networks:
      - app-network

  # PostgreSQL database
  database:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=appdb
      - POSTGRES_USER=appuser
      - POSTGRES_PASSWORD_FILE=/run/secrets/db_password
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - app-network
    secrets:
      - db_password

  # Redis cache
  cache:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - cache-data:/data
    networks:
      - app-network

volumes:
  db-data:
    driver: local
  cache-data:
    driver: local

networks:
  app-network:
    driver: bridge

secrets:
  db_password:
    external: true
```

## Step 4: Configure Stack Settings

Before deploying, configure additional settings:

- **Stack name** - give your stack a meaningful name (e.g., `my-webapp`)
- **Environment variables** - add `.env` variables or define them inline
- **Access control** - restrict stack management to specific teams (BE only)

## Step 5: Deploy the Stack

1. Click **Deploy the stack**
2. Portainer pulls the images and creates all services
3. Monitor the deployment output in the logs panel

## Step 6: Verify the Deployment

After deployment:

1. Click on the stack name to view all services
2. Check that each container shows a **Running** status
3. Click on individual containers to view logs
4. Use the **Console** feature to exec into containers if needed

## Managing the Stack

From the stack detail page you can:

- **Update** the stack by modifying the Compose file
- **Stop/Start** all services at once
- **Remove** the entire stack and optionally remove volumes
- **Duplicate** the stack to another environment

## Environment Variables in Stacks

Use the **Environment variables** section in Portainer to inject secrets without hardcoding them:

```yaml
services:
  api:
    image: myapi:latest
    environment:
      - DB_PASSWORD=${DB_PASSWORD}   # Injected from Portainer env vars
      - API_KEY=${API_KEY}
```

In the Portainer UI, add `DB_PASSWORD` and `API_KEY` as environment variable entries before deploying.

## Troubleshooting

- **Image pull errors** - verify registry credentials are configured in Portainer
- **Port conflicts** - check that host ports are not already in use
- **Volume mount errors** - ensure the host path exists or use named volumes
- **Network errors** - confirm network names are unique within the environment

## Conclusion

Portainer Stacks make it straightforward to deploy and manage complex multi-service applications. By combining Docker Compose syntax with Portainer's visual interface, you get the best of both worlds: infrastructure-as-code for repeatability and a GUI for easy day-to-day management. Start with a simple Compose file and progressively add services as your application grows.
