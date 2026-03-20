# How to Deploy StackStorm via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, StackStorm, Automation, Docker, DevOps

Description: Deploy StackStorm event-driven automation platform using Portainer for IT operations automation and ChatOps.

## Introduction

StackStorm (ST2) is an event-driven automation platform for DevOps and IT operations. It connects sensors (event sources) to actions (automated responses) via rules, with support for workflows, packs (integrations), and ChatOps via Slack/Teams.

## Prerequisites

- Portainer installed with Docker
- At least 2 GB RAM

## Step 1: Create the Stack in Portainer

Navigate to **Stacks** > **Add Stack**:

```yaml
# docker-compose.yml - StackStorm
version: "3.8"

services:
  stackstorm:
    image: stackstorm/stackstorm:3.8.1
    container_name: stackstorm
    restart: unless-stopped
    ports:
      - "443:443"
      - "80:80"
    volumes:
      - stackstorm_packs:/opt/stackstorm/packs
      - stackstorm_configs:/opt/stackstorm/configs
      - stackstorm_log:/var/log/st2
    environment:
      - ST2_AUTH_USERNAME=st2admin
      - ST2_AUTH_PASSWORD=${ST2_PASSWORD}
      - RABBITMQ_DEFAULT_USER=st2
      - RABBITMQ_DEFAULT_PASS=${RABBITMQ_PASSWORD}
    depends_on:
      - mongo
      - redis
      - rabbitmq
    networks:
      - st2_net

  mongo:
    image: mongo:7.0
    container_name: st2_mongo
    restart: unless-stopped
    volumes:
      - st2_mongo_data:/data/db
    networks:
      - st2_net

  redis:
    image: redis:7-alpine
    container_name: st2_redis
    restart: unless-stopped
    networks:
      - st2_net

  rabbitmq:
    image: rabbitmq:3.13-management-alpine
    container_name: st2_rabbitmq
    restart: unless-stopped
    environment:
      - RABBITMQ_DEFAULT_USER=st2
      - RABBITMQ_DEFAULT_PASS=${RABBITMQ_PASSWORD}
    networks:
      - st2_net

volumes:
  stackstorm_packs:
  stackstorm_configs:
  stackstorm_log:
  st2_mongo_data:

networks:
  st2_net:
    driver: bridge
```

## Step 2: Set Environment Variables in Portainer

```
ST2_PASSWORD=your-st2-admin-password
RABBITMQ_PASSWORD=your-rabbitmq-password
```

## Step 3: Access the StackStorm UI

Open `https://<host>` and log in with `st2admin` / your ST2 password.

## Step 4: Use the ST2 CLI

```bash
# Authenticate
docker exec stackstorm st2 auth st2admin -p your-st2-admin-password

# List available packs
docker exec stackstorm st2 pack list

# Run an action
docker exec stackstorm st2 run core.local cmd="echo Hello from StackStorm"

# List triggers
docker exec stackstorm st2 trigger list
```

## Step 5: Install a Pack

```bash
# Install the Slack pack for ChatOps
docker exec stackstorm st2 pack install slack

# Configure the pack
docker exec stackstorm st2 pack config slack
```

## Step 6: Create a Rule

```bash
# Create a rule file
cat > /tmp/my_rule.yaml << 'EOF'
name: "on_timer_hello"
pack: "default"
description: "Say hello every minute"
trigger:
  type: "core.st2.IntervalTimer"
  parameters:
    delta: 60
    unit: "seconds"
criteria: {}
action:
  ref: "core.local"
  parameters:
    cmd: "echo Hello from rule at $(date)"
enabled: true
EOF

# Deploy the rule
docker cp /tmp/my_rule.yaml stackstorm:/opt/stackstorm/rules/
docker exec stackstorm st2 rule create /opt/stackstorm/rules/my_rule.yaml
```

## Conclusion

StackStorm's event-driven model works via three components: Sensors (detect events), Triggers (events that rules react to), and Actions (automated tasks). Rules wire triggers to actions with optional criteria filters. Packs bundle sensors, triggers, actions, and rules for specific services (AWS, PagerDuty, Slack, JIRA). StackStorm uses MongoDB for state, RabbitMQ for messaging, and Redis for result caching.
