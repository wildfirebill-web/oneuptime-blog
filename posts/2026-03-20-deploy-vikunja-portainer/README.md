# How to Deploy Vikunja (Task Manager) via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Vikunja, Task Manager, Docker, Self-Hosted

Description: Deploy Vikunja open-source to-do app and project manager using Portainer as a self-hosted alternative to Todoist and Trello.

## Introduction

Vikunja is an open-source to-do app and project manager with lists, kanban boards, Gantt charts, and team collaboration. It provides a REST API with a web frontend and supports CALDAV for calendar integration.

## Prerequisites

- Portainer installed with Docker

## Step 1: Create the Stack in Portainer

Navigate to **Stacks** > **Add Stack**:

```yaml
# docker-compose.yml - Vikunja

version: "3.8"

services:
  vikunja:
    image: vikunja/vikunja:0.23.0
    container_name: vikunja
    restart: unless-stopped
    ports:
      - "3456:3456"
    volumes:
      - vikunja_files:/app/vikunja/files
    environment:
      VIKUNJA_DATABASE_HOST: vikunja_postgres
      VIKUNJA_DATABASE_PASSWORD: ${DB_PASSWORD}
      VIKUNJA_DATABASE_TYPE: postgres
      VIKUNJA_DATABASE_USER: vikunja
      VIKUNJA_DATABASE_DATABASE: vikunja
      VIKUNJA_SERVICE_JWTSECRET: ${JWT_SECRET}
      VIKUNJA_SERVICE_FRONTENDURL: http://${VIKUNJA_DOMAIN}:3456
      VIKUNJA_MAILER_ENABLED: "false"
    depends_on:
      vikunja_postgres:
        condition: service_healthy
    networks:
      - vikunja_net

  vikunja_postgres:
    image: postgres:16-alpine
    container_name: vikunja_postgres
    restart: unless-stopped
    volumes:
      - vikunja_postgres_data:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: vikunja
      POSTGRES_USER: vikunja
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U vikunja"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - vikunja_net

volumes:
  vikunja_files:
  vikunja_postgres_data:

networks:
  vikunja_net:
    driver: bridge
```

## Step 2: Set Environment Variables in Portainer

```text
DB_PASSWORD=your-postgres-password
JWT_SECRET=your-jwt-secret-min-32-chars
VIKUNJA_DOMAIN=vikunja.yourdomain.com
```

## Step 3: Access Vikunja

Open `http://<host>:3456` and register a new user account.

## Step 4: Use the REST API

```bash
# Get an auth token
curl -X POST http://localhost:3456/api/v1/login \
  -H 'Content-Type: application/json' \
  -d '{"username": "your-username", "password": "your-password"}'

# Create a project (list)
curl -X POST http://localhost:3456/api/v1/projects \
  -H 'Authorization: Bearer <token>' \
  -H 'Content-Type: application/json' \
  -d '{"title": "My Project"}'

# Create a task
curl -X PUT http://localhost:3456/api/v1/projects/1/tasks \
  -H 'Authorization: Bearer <token>' \
  -H 'Content-Type: application/json' \
  -d '{"title": "My First Task", "due_date": "2024-12-31T00:00:00Z"}'

# List tasks in a project
curl http://localhost:3456/api/v1/projects/1/tasks \
  -H 'Authorization: Bearer <token>'
```

## Step 5: CALDAV Integration

```bash
# Vikunja supports CALDAV for calendar clients
# Connect at: http://localhost:3456/dav/principals/<username>/

# For Thunderbird/Nextcloud Tasks, use:
# Server: http://localhost:3456/dav/
# Username: your-username
# Password: your-password
```

## Conclusion

Vikunja combines the `vikunja/vikunja` image (which includes both the API backend and the web frontend) into a single container. The `VIKUNJA_SERVICE_FRONTENDURL` must match your actual URL for links in notifications to work correctly. Enable SMTP via `VIKUNJA_MAILER_*` environment variables for email notifications and password reset functionality.
