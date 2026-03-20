# How to Create a Stack from the Web Editor in Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Stacks, Docker Compose, DevOps

Description: Learn how to create and deploy Docker Compose stacks directly from Portainer's built-in web editor with environment variable support.

## Introduction

Portainer's web editor allows you to paste or type Docker Compose YAML directly in the browser and deploy it as a stack - no file uploads or Git repositories needed. This is the fastest way to get a multi-container application running in Portainer and is ideal for quick deployments, testing configurations, and learning Docker Compose. The editor includes syntax highlighting and immediate environment variable substitution.

## Prerequisites

- Portainer installed with a connected Docker environment
- Basic understanding of Docker Compose syntax

## What is a Portainer Stack

A Portainer stack is a Docker Compose deployment managed through Portainer. It groups all containers, networks, and volumes defined in a Compose file into a single named unit that can be started, stopped, updated, or removed together.

## Step 1: Navigate to Stacks

1. Log into Portainer.
2. Select your environment (local Docker or a remote endpoint).
3. Navigate to **Stacks** in the left menu.
4. Click **Add stack**.

## Step 2: Configure the Stack

1. Enter a **Name** for the stack (lowercase, alphanumeric, hyphens allowed): `my-web-app`
2. Select **Web editor** as the build method (should be selected by default).
3. The editor pane opens for you to enter your Compose YAML.

## Step 3: Enter a Docker Compose Configuration

Paste a complete Compose file in the editor:

```yaml
# Full-stack web application

version: "3.8"

services:
  # Nginx reverse proxy - receives external traffic
  nginx:
    image: nginx:alpine
    restart: unless-stopped
    ports:
      - "80:80"
    volumes:
      - nginx_conf:/etc/nginx/conf.d
    networks:
      - frontend
      - backend
    depends_on:
      - api

  # Node.js API server
  api:
    image: node:18-alpine
    restart: unless-stopped
    working_dir: /app
    command: ["node", "server.js"]
    volumes:
      - app_data:/app
    networks:
      - backend
      - data
    environment:
      - NODE_ENV=${NODE_ENV:-production}
      - DB_HOST=postgres
      - DB_PORT=5432
      - DB_NAME=${DB_NAME}
      - DB_PASSWORD=${DB_PASSWORD}
      - REDIS_URL=redis://redis:6379

  # PostgreSQL database
  postgres:
    image: postgres:15-alpine
    restart: unless-stopped
    networks:
      - data
    environment:
      - POSTGRES_DB=${DB_NAME}
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data

  # Redis cache
  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: ["redis-server", "--appendonly", "yes"]
    networks:
      - data
    volumes:
      - redis_data:/data

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true
  data:
    driver: bridge
    internal: true

volumes:
  nginx_conf:
  app_data:
  postgres_data:
  redis_data:
```

## Step 4: Add Environment Variables

Below the editor, you'll find the **Environment variables** section:

1. Click **Add an environment variable** for each variable.
2. Enter the key and value:

```text
DB_NAME        myapp_production
DB_PASSWORD    supersecretpassword
NODE_ENV       production
```

Or use the **Advanced mode** to paste a `.env` file format:

```text
DB_NAME=myapp_production
DB_PASSWORD=supersecretpassword
NODE_ENV=production
```

## Step 5: Deploy the Stack

1. Review the configuration.
2. Click **Deploy the stack**.
3. Portainer pulls images, creates networks and volumes, and starts containers.
4. You'll see the stack appear in the Stacks list with all container statuses.

## Step 6: Verify the Deployment

After deployment, in Portainer:
1. Click the stack name to see all containers.
2. Check each container is in **Running** state.
3. Click a container to view its logs if there are issues.

Via CLI:
```bash
# List stack containers:
docker stack ps my-web-app   # Swarm
# or for Compose:
docker ps --filter "label=com.docker.compose.project=my-web-app"

# View logs:
docker logs my-web-app_api_1

# Test the deployed service:
curl http://localhost/health
```

## Step 7: Update the Stack via Web Editor

To modify the stack:
1. Navigate to **Stacks** → click the stack name.
2. The editor shows the current Compose content.
3. Make changes directly in the editor.
4. Click **Update the stack**.

Portainer will apply only the changed services (rolling update).

## Conclusion

The web editor is the quickest way to deploy multi-container applications in Portainer. Paste your Docker Compose YAML, set environment variables in the UI, and click Deploy. All containers, networks, and volumes are created as a single managed unit. For production deployments that need version control and auditability, consider deploying from a Git repository instead - but the web editor is ideal for rapid iteration, testing, and learning Docker Compose without any external tooling.
