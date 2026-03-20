# How to Deploy Hasura GraphQL Engine via Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Hasura, GraphQL, Docker, PostgreSQL

Description: Deploy Hasura GraphQL Engine using Portainer to instantly generate a real-time GraphQL API over your PostgreSQL database.

## Introduction

Hasura GraphQL Engine connects to a PostgreSQL database and auto-generates a full-featured GraphQL API with queries, mutations, and subscriptions. It supports row-level authorization (permissions), remote schemas, event triggers, and scheduled triggers.

## Prerequisites

- Portainer installed with Docker

## Step 1: Create the Stack in Portainer

Navigate to **Stacks** > **Add Stack**:

```yaml
# docker-compose.yml - Hasura GraphQL Engine

version: "3.8"

services:
  hasura:
    image: hasura/graphql-engine:v2.40.0
    container_name: hasura
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      - HASURA_GRAPHQL_DATABASE_URL=postgres://hasura:${DB_PASSWORD}@hasura_postgres:5432/hasura
      - HASURA_GRAPHQL_ENABLE_CONSOLE=true
      - HASURA_GRAPHQL_DEV_MODE=false
      - HASURA_GRAPHQL_ENABLED_LOG_TYPES=startup,http-log,webhook-log,websocket-log,query-log
      - HASURA_GRAPHQL_ADMIN_SECRET=${HASURA_ADMIN_SECRET}
      - HASURA_GRAPHQL_JWT_SECRET=${HASURA_JWT_SECRET}
    depends_on:
      hasura_postgres:
        condition: service_healthy
    networks:
      - hasura_net

  hasura_postgres:
    image: postgres:16-alpine
    container_name: hasura_postgres
    restart: unless-stopped
    volumes:
      - hasura_postgres_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=hasura
      - POSTGRES_USER=hasura
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U hasura"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - hasura_net

volumes:
  hasura_postgres_data:

networks:
  hasura_net:
    driver: bridge
```

## Step 2: Set Environment Variables in Portainer

```text
DB_PASSWORD=your-postgres-password
HASURA_ADMIN_SECRET=your-admin-secret-min-32-chars
HASURA_JWT_SECRET={"type":"HS256","key":"your-jwt-secret-min-32-chars"}
```

## Step 3: Access the Hasura Console

Open `http://<host>:8080/console` and enter your `HASURA_ADMIN_SECRET`.

## Step 4: Track Tables and Run GraphQL Queries

In the Hasura Console:
1. Go to **Data** > **Track all** to expose your tables as GraphQL types
2. Set up relationships between tables

```graphql
# Query example
query GetUsers {
  users(limit: 10, order_by: {created_at: desc}) {
    id
    name
    email
    created_at
  }
}

# Mutation example
mutation InsertUser {
  insert_users_one(object: {name: "Alice", email: "alice@example.com"}) {
    id
    created_at
  }
}

# Subscription (real-time)
subscription WatchUsers {
  users(limit: 5, order_by: {created_at: desc}) {
    id
    name
  }
}
```

## Step 5: Apply Metadata with the CLI

```bash
# Install Hasura CLI
curl -L https://github.com/hasura/graphql-engine/raw/stable/cli/get.sh | bash

# Initialize a Hasura project
hasura init my-project --endpoint http://localhost:8080 \
  --admin-secret your-admin-secret

# Export current metadata
cd my-project
hasura metadata export

# Apply metadata to a new instance
hasura metadata apply
```

## Step 6: Set Row-Level Permissions

In the Hasura Console under **Data** > **<table>** > **Permissions**:

```json
// Allow users to only read their own rows
{
  "user_id": {
    "_eq": "X-Hasura-User-Id"
  }
}
```

## Conclusion

Hasura auto-generates a GraphQL API the moment you track your tables - no resolvers to write. The `HASURA_GRAPHQL_JWT_SECRET` enables token-based auth: include a JWT with `x-hasura-user-id` and `x-hasura-role` claims and Hasura enforces permissions automatically. Disable `HASURA_GRAPHQL_DEV_MODE` and `HASURA_GRAPHQL_ENABLE_CONSOLE` in production-facing deployments.
