# How to Deploy Wekan (Kanban Board) via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Wekan,Kanban,Portainer,Docker,Project Management,Self-Hosted

Description: Deploy Wekan open-source Kanban board via Portainer with MongoDB persistence for team task management and project tracking.

---

This guide covers deploying this self-hosted productivity application via Portainer with persistent data storage and proper configuration.

## Deploy via Portainer Stack

Navigate to **Stacks > Add Stack** in Portainer and use the following configuration:

```yaml
version: "3.8"
services:
  app:
    image: wekan-kanban
    environment:
      - DATABASE_URL=postgres://app:password@postgres:5432/appdb
    volumes:
      - app-data:/app/data
    ports:
      - "80:80"
    depends_on:
      postgres:
        condition: service_healthy
    restart: unless-stopped
    networks:
      - app-net

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: app
      POSTGRES_PASSWORD: password
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped
    networks:
      - app-net

volumes:
  app-data:
  postgres-data:

networks:
  app-net:
    driver: bridge
```

## Configuration

After deployment, access the application at `http://host:80` and complete the initial setup:

1. Create the first admin user
2. Configure your workspace or organization settings
3. Invite team members via the admin panel
4. Configure email notifications (SMTP settings)

## Key Features

This application provides:

- **Kanban boards / Project tracking** - visual workflow management
- **Team collaboration** - assign tasks and track progress
- **Labels and categories** - organize work by type or priority
- **Due dates and deadlines** - time-based task management
- **Comments and attachments** - rich context on each task

## Backup and Restore

Backup the application data:

```bash
# Backup PostgreSQL database

docker exec postgres_container pg_dump -U app appdb > backup-$(date +%Y%m%d).sql

# Backup application files
docker run --rm \
  -v app-data:/data:ro \
  -v /opt/backups:/backups \
  alpine tar czf "/backups/app-data-$(date +%Y%m%d).tar.gz" /data
```

## Summary

This self-hosted productivity tool deployed via Portainer gives your team a private, data-owned alternative to SaaS project management platforms. Portainer handles the container lifecycle, and PostgreSQL provides reliable persistent storage for all project data.
