# How to Deploy Taiga (Project Management) via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Taiga, Project Management, Docker, Self-Hosted

Description: Deploy Taiga open-source agile project management platform using Portainer for Scrum and Kanban workflows.

## Introduction

Taiga is an open-source agile project management platform supporting Scrum, Kanban, and issue tracking. It features a clean UI, epics, user stories, sprints, backlogs, and wikis. This guide deploys Taiga using the official Docker Compose setup.

## Prerequisites

- Portainer installed with Docker
- At least 2 GB RAM
- A domain name for `TAIGA_SITES_DOMAIN`

## Step 1: Create the Stack in Portainer

Navigate to **Stacks** > **Add Stack**:

```yaml
# docker-compose.yml - Taiga
version: "3.8"

x-environment: &default-back-environment
  POSTGRES_DB: taiga
  POSTGRES_USER: taiga
  POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
  POSTGRES_HOST: taiga-db
  TAIGA_SECRET_KEY: ${TAIGA_SECRET_KEY}
  TAIGA_SITES_SCHEME: http
  TAIGA_SITES_DOMAIN: ${TAIGA_DOMAIN}
  TAIGA_SUBPATH: ""

services:
  taiga-db:
    image: postgres:15-alpine
    container_name: taiga_db
    restart: unless-stopped
    volumes:
      - taiga_db_data:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: taiga
      POSTGRES_USER: taiga
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U taiga"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - taiga_net

  taiga-back:
    image: taigaio/taiga-back:6.8.1
    container_name: taiga_back
    restart: unless-stopped
    volumes:
      - taiga_static:/taiga-back/static
      - taiga_media:/taiga-back/media
    environment:
      <<: *default-back-environment
      ENABLE_TELEMETRY: "False"
    depends_on:
      taiga-db:
        condition: service_healthy
      taiga-events-rabbitmq:
        condition: service_started
    networks:
      - taiga_net

  taiga-async:
    image: taigaio/taiga-back:6.8.1
    container_name: taiga_async
    restart: unless-stopped
    command: ["/taiga-back/docker/async_entrypoint.sh"]
    volumes:
      - taiga_static:/taiga-back/static
      - taiga_media:/taiga-back/media
    environment:
      <<: *default-back-environment
    depends_on:
      - taiga-back
    networks:
      - taiga_net

  taiga-front:
    image: taigaio/taiga-front:6.8.1
    container_name: taiga_front
    restart: unless-stopped
    environment:
      TAIGA_URL: http://${TAIGA_DOMAIN}
      TAIGA_WEBSOCKETS_URL: ws://${TAIGA_DOMAIN}
    networks:
      - taiga_net

  taiga-events-rabbitmq:
    image: rabbitmq:3.13-management-alpine
    container_name: taiga_rabbitmq
    restart: unless-stopped
    environment:
      RABBITMQ_ERLANG_COOKIE: ${RABBITMQ_ERLANG_COOKIE}
      RABBITMQ_DEFAULT_USER: taiga
      RABBITMQ_DEFAULT_PASS: ${RABBITMQ_PASSWORD}
    networks:
      - taiga_net

  taiga-gateway:
    image: nginx:1.25-alpine
    container_name: taiga_gateway
    restart: unless-stopped
    ports:
      - "80:80"
    volumes:
      - ./taiga.conf:/etc/nginx/conf.d/default.conf:ro
      - taiga_static:/taiga/static:ro
      - taiga_media:/taiga/media:ro
    depends_on:
      - taiga-front
      - taiga-back
    networks:
      - taiga_net

volumes:
  taiga_db_data:
  taiga_static:
  taiga_media:

networks:
  taiga_net:
    driver: bridge
```

## Step 2: Set Environment Variables in Portainer

```
POSTGRES_PASSWORD=your-postgres-password
TAIGA_SECRET_KEY=your-secret-key-min-32-chars
TAIGA_DOMAIN=taiga.yourdomain.com
RABBITMQ_ERLANG_COOKIE=your-erlang-cookie
RABBITMQ_PASSWORD=your-rabbitmq-password
```

## Step 3: Access Taiga

Open `http://<host>` and log in with default credentials:
- Username: `admin`
- Password: `123123`

Change the password immediately after login.

## Step 4: Create a Project

1. Click **New Project**
2. Choose **Scrum** or **Kanban**
3. Add team members, create user stories, and organize sprints

## Conclusion

Taiga's architecture separates backend (Django), frontend (Angular), async workers, and real-time events (via RabbitMQ). All user-uploaded media is stored in the `taiga_media` volume. The nginx gateway serves static assets and proxies API requests. For production, configure SMTP via `DEFAULT_FROM_EMAIL` and `EMAIL_*` environment variables in the backend service.
