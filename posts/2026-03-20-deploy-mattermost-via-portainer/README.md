# How to Deploy Mattermost via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Mattermost, Team Chat, Docker, Self-Hosting, PostgreSQL, Slack Alternative

Description: Learn how to deploy Mattermost, the open-source Slack alternative, via Portainer with a PostgreSQL backend and persistent configuration storage.

---

Mattermost is an enterprise-grade messaging platform you can host on your own infrastructure. It supports channels, direct messages, file sharing, and integrations. Portainer makes deploying and managing the two-container stack easy.

## Prerequisites

- Portainer running
- At least 1GB RAM
- A domain for the Mattermost URL

## Compose Stack

```yaml
version: "3.8"

services:
  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_DB: mattermost
      POSTGRES_USER: mmuser
      POSTGRES_PASSWORD: mmpass           # Change this
    volumes:
      - mm_db:/var/lib/postgresql/data

  mattermost:
    image: mattermost/mattermost-team-edition:latest
    restart: unless-stopped
    depends_on:
      - postgres
    ports:
      - "8065:8065"
    environment:
      MM_SQLSETTINGS_DRIVERNAME: postgres
      MM_SQLSETTINGS_DATASOURCE: postgres://mmuser:mmpass@postgres:5432/mattermost?sslmode=disable
      MM_BLEVESETTINGS_INDEXDIR: /mattermost/bleve-indexes
      MM_SERVICESETTINGS_SITEURL: https://chat.example.com
    volumes:
      - mm_config:/mattermost/config
      - mm_data:/mattermost/data
      - mm_logs:/mattermost/logs
      - mm_plugins:/mattermost/plugins
      - mm_client_plugins:/mattermost/client/plugins

volumes:
  mm_db:
  mm_config:
  mm_data:
  mm_logs:
  mm_plugins:
  mm_client_plugins:
```

## Deploying

1. In Portainer go to **Stacks > Add Stack**.
2. Name it `mattermost`.
3. Update `MM_SQLSETTINGS_DATASOURCE` with your password and `MM_SERVICESETTINGS_SITEURL` with your domain.
4. Click **Deploy the stack**.

Open `http://<host>:8065` to complete setup and create the first team.

## Email Notifications

Configure SMTP for email invites and notifications under **System Console > Environment > SMTP**:

```
SMTP Server: smtp.example.com
SMTP Port:   587
SMTP Username: no-reply@example.com
Enable SMTP Authentication: true
```

## Monitoring

Use OneUptime to monitor `http://<host>:8065/api/v4/system/ping`. Mattermost returns `{"status":"OK"}` when healthy. Alert on any degraded status since chat downtime disrupts team communication directly.
