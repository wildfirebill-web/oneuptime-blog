# How to Deploy Rocket.Chat via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Rocket.Chat, Team Chat, Docker, Self-Hosting, MongoDB, Slack Alternative

Description: Learn how to deploy Rocket.Chat, the open-source team communication platform, via Portainer with a MongoDB backend and persistent data volumes.

---

Rocket.Chat is a feature-rich open-source messaging platform with support for channels, voice/video calls, omnichannel customer support, and a robust API. It uses MongoDB as its database. Portainer simplifies the two-service stack management.

## Prerequisites

- Portainer running
- At least 2GB RAM (MongoDB + Rocket.Chat are both memory hungry)
- A domain for the `ROOT_URL` setting

## Compose Stack

```yaml
version: "3.8"

services:
  mongo:
    image: mongo:6.0
    restart: unless-stopped
    # Enable replica set mode - required by Rocket.Chat
    command: mongod --oplogSize 128 --replSet rs0
    volumes:
      - mongo_data:/data/db
      - mongo_dump:/dump

  mongo-init-replica:
    image: mongo:6.0
    # One-time job to initialize the replica set
    command: >
      bash -c "sleep 10 && mongosh --host mongo --eval \"rs.initiate({_id: 'rs0', members: [{_id: 0, host: 'mongo:27017'}]})\""
    depends_on:
      - mongo

  rocketchat:
    image: registry.rocket.chat/rocketchat/rocket.chat:latest
    restart: unless-stopped
    depends_on:
      - mongo
    ports:
      - "3102:3000"
    environment:
      MONGO_URL: mongodb://mongo:27017/rocketchat?replicaSet=rs0
      MONGO_OPLOG_URL: mongodb://mongo:27017/local?replicaSet=rs0
      ROOT_URL: https://chat.example.com
      PORT: 3000
      DEPLOY_PLATFORM: docker

volumes:
  mongo_data:
  mongo_dump:
```

## Deploying

1. In Portainer go to **Stacks > Add Stack**.
2. Name it `rocketchat`.
3. Update `ROOT_URL` to your actual domain.
4. Click **Deploy the stack**.

The `mongo-init-replica` service runs once and exits, which Portainer will show as "Exited (0)" — this is normal.

Open `http://<host>:3102` and complete the setup wizard.

## Integrations

Rocket.Chat integrates with hundreds of services via webhooks and native integrations. Common ones include:

- **GitHub/GitLab**: Post commit and PR notifications to channels
- **Jira**: Link issues to messages
- **Zapier/n8n**: Automate workflows

## Monitoring

Use OneUptime to monitor `http://<host>:3102/api/v1/info`. Rocket.Chat returns version and deployment info when healthy. Alert on any non-200 responses to catch issues before your team notices.
