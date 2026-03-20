# How to Deploy Wekan (Kanban Board) via Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Wekan, Kanban, Docker, Self-Hosted

Description: Deploy Wekan open-source kanban board using Portainer as a self-hosted Trello alternative.

## Introduction

Wekan is an open-source, self-hosted Kanban board built with Meteor and MongoDB. It supports boards, lists, cards, labels, checklists, due dates, and file attachments - a Trello-compatible alternative for teams.

## Prerequisites

- Portainer installed with Docker

## Step 1: Create the Stack in Portainer

Navigate to **Stacks** > **Add Stack**:

```yaml
# docker-compose.yml - Wekan

version: "3.8"

services:
  wekan:
    image: ghcr.io/wekan/wekan:v7.55
    container_name: wekan
    restart: unless-stopped
    ports:
      - "8080:8080"
    volumes:
      - wekan_files:/data
    environment:
      - MONGO_URL=mongodb://wekan_mongo:27017/wekan
      - ROOT_URL=http://${WEKAN_DOMAIN}:8080
      - WITH_API=true
      - BROWSER_POLICY_ENABLED=true
      - TRUSTED_URL=http://${WEKAN_DOMAIN}:8080
      - RICHER_CARD_COMMENT_EDITOR=false
      - CARD_OPENED_WEBHOOK_ENABLED=false
      - NODE_ENV=production
    depends_on:
      - wekan_mongo
    networks:
      - wekan_net

  wekan_mongo:
    image: mongo:7.0
    container_name: wekan_mongo
    restart: unless-stopped
    volumes:
      - wekan_mongo_data:/data/db
    command: mongod --oplogSize 128
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 30s
      timeout: 10s
      retries: 5
    networks:
      - wekan_net

volumes:
  wekan_files:
  wekan_mongo_data:

networks:
  wekan_net:
    driver: bridge
```

## Step 2: Set Environment Variables in Portainer

```text
WEKAN_DOMAIN=wekan.yourdomain.com
```

## Step 3: Access Wekan

Open `http://<host>:8080` and register. The first registered user becomes the admin.

## Step 4: Create a Board

1. Click **Add Board**
2. Add lists (columns) such as "Backlog", "In Progress", "Done"
3. Create cards within lists
4. Set labels, due dates, and assign members

## Step 5: Use the REST API

```bash
# Login to get a token
TOKEN=$(curl -s -X POST http://localhost:8080/users/login \
  -H 'Content-Type: application/json' \
  -d '{"username": "admin", "password": "your-password"}' | \
  python3 -c "import sys,json; d=json.load(sys.stdin); print(d['token'])")

USER_ID=$(curl -s -X POST http://localhost:8080/users/login \
  -H 'Content-Type: application/json' \
  -d '{"username": "admin", "password": "your-password"}' | \
  python3 -c "import sys,json; d=json.load(sys.stdin); print(d['id'])")

# List boards
curl http://localhost:8080/api/users/${USER_ID}/boards \
  -H "Authorization: Bearer ${TOKEN}"

# Create a card (need list ID from board)
curl -X POST "http://localhost:8080/api/boards/<board-id>/lists/<list-id>/cards" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H 'Content-Type: application/json' \
  -d '{"title": "New Task", "authorId": "'${USER_ID}'"}'
```

## Step 6: Back Up Wekan Data

```bash
# Backup MongoDB
docker exec wekan_mongo mongodump --out /tmp/wekan_backup
docker cp wekan_mongo:/tmp/wekan_backup ./wekan_backup_$(date +%Y%m%d)

# Restore
docker cp ./wekan_backup_20240101 wekan_mongo:/tmp/restore_backup
docker exec wekan_mongo mongorestore /tmp/restore_backup
```

## Conclusion

Wekan requires `ROOT_URL` to be set to the exact URL used to access the application - incorrect values cause login and redirect issues. The `--oplogSize 128` MongoDB flag limits the oplog to 128 MB for single-node deployments. For production, configure SMTP via `MAIL_URL=smtp://user:pass@host:port` and `MAIL_FROM=noreply@yourdomain.com` for email notifications.
