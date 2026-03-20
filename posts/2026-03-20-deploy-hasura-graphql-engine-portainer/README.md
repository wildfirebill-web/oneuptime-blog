# How to Deploy Hasura GraphQL Engine via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Hasura, GraphQL, Portainer, Docker, PostgreSQL, API, Backend

Description: Deploy Hasura GraphQL Engine via Portainer to instantly generate a powerful GraphQL and REST API from your PostgreSQL database with real-time subscriptions and row-level security.

---

Hasura connects to your PostgreSQL database and instantly generates a GraphQL API with queries, mutations, and subscriptions. Deploying it via Portainer alongside PostgreSQL gives you a fully functional API backend in minutes.

## Deploy Hasura Stack

```yaml
# hasura-stack.yml
version: "3.8"
services:
  hasura:
    image: hasura/graphql-engine:v2.38.0
    environment:
      # PostgreSQL connection string
      HASURA_GRAPHQL_DATABASE_URL: postgres://hasura:${DB_PASSWORD:-hasura_password}@postgres:5432/hasura
      # Enable the console (disable in production or protect with auth)
      HASURA_GRAPHQL_ENABLE_CONSOLE: "true"
      # Admin secret — protect the console and API
      HASURA_GRAPHQL_ADMIN_SECRET: ${ADMIN_SECRET:-change-this-secret}
      # JWT configuration for user auth
      HASURA_GRAPHQL_JWT_SECRET: '{"type":"RS256","jwk_url":"https://auth.example.com/.well-known/jwks.json"}'
      # Development-only settings
      HASURA_GRAPHQL_DEV_MODE: "false"
      HASURA_GRAPHQL_ENABLED_LOG_TYPES: startup, http-log, webhook-log, websocket-log
    ports:
      - "8080:8080"
    depends_on:
      postgres:
        condition: service_healthy
    restart: unless-stopped
    networks:
      - hasura-net

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: hasura
      POSTGRES_USER: hasura
      POSTGRES_PASSWORD: ${DB_PASSWORD:-hasura_password}
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U hasura"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped
    networks:
      - hasura-net

volumes:
  postgres-data:

networks:
  hasura-net:
    driver: bridge
```

## Access the Hasura Console

Open `http://host:8080/console` and enter your admin secret. From the console:

1. **Data** — track your tables and views to expose them via GraphQL
2. **GraphiQL** — interactive GraphQL IDE for testing queries
3. **Events** — configure event triggers on database changes
4. **Remote Schemas** — merge external GraphQL APIs

## Sample GraphQL Queries

Once tables are tracked, Hasura generates CRUD operations automatically:

```graphql
# Query all users
query GetUsers {
  users {
    id
    email
    created_at
    profile {
      name
      avatar_url
    }
  }
}

# Insert a new post
mutation CreatePost($title: String!, $content: String!, $author_id: uuid!) {
  insert_posts_one(object: {
    title: $title
    content: $content
    author_id: $author_id
  }) {
    id
    created_at
  }
}

# Real-time subscription to new messages
subscription OnNewMessage($channel_id: uuid!) {
  messages(
    where: { channel_id: { _eq: $channel_id } }
    order_by: { created_at: desc }
    limit: 10
  ) {
    id
    content
    sender { name }
  }
}
```

## Row-Level Security

Define permissions in the Hasura console for each role:

```json
// Allow users to only see their own data
{
  "role": "user",
  "permission": {
    "columns": ["id", "email", "profile"],
    "filter": {
      "id": { "_eq": "X-Hasura-User-Id" }
    }
  }
}
```

## REST Endpoint Generation

Hasura can expose your queries as REST endpoints:

1. Go to **REST** in the console
2. Name your endpoint
3. Select the saved query/mutation to expose

```bash
# Now callable as a REST endpoint
curl "http://hasura:8080/api/rest/v1/users?id=uuid-here" \
  -H "Authorization: Bearer user-jwt-token"
```

## Summary

Hasura deployed via Portainer provides an instant GraphQL and REST API layer over PostgreSQL. With row-level security, real-time subscriptions, and event triggers, Hasura is a powerful backend accelerator that eliminates the need to hand-write CRUD API endpoints.
