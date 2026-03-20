# How to Override Stack Configuration for Different Environments

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker Compose, Environment Override, Multi-Environment, Configuration, DevOps

Description: Use Portainer's environment variables, Git branch deployments, and Docker Compose override files to maintain a single stack definition that works across development, staging, and production environments.

---

Managing separate stack files for each environment leads to drift and maintenance overhead. A better approach is a single base stack with environment-specific overrides. Portainer supports several patterns for this.

## Method 1: Environment Variables in Portainer

Use Portainer's built-in environment variable management to override stack values per environment:

```yaml
# Base stack — same file deployed to all environments
version: "3.8"
services:
  webapp:
    image: myapp:${APP_VERSION:-latest}
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - LOG_LEVEL=${LOG_LEVEL:-info}
      - REPLICAS=${REPLICAS:-1}
    deploy:
      replicas: ${REPLICAS:-1}
    restart: ${RESTART_POLICY:-unless-stopped}
```

In Portainer, set different values per environment:

**Production environment variables:**
```
APP_VERSION=1.2.3
DATABASE_URL=postgres://prod-db:5432/mydb
LOG_LEVEL=warn
REPLICAS=3
RESTART_POLICY=always
```

**Staging environment variables:**
```
APP_VERSION=1.3.0-beta
DATABASE_URL=postgres://staging-db:5432/mydb
LOG_LEVEL=debug
REPLICAS=1
RESTART_POLICY=unless-stopped
```

## Method 2: Docker Compose Override Files

Use a base file with environment-specific override files:

```yaml
# docker-compose.yml (base — checked into Git)
version: "3.8"
services:
  webapp:
    image: myapp:1.2.3
    environment:
      - NODE_ENV=production

  database:
    image: postgres:16-alpine
    volumes:
      - db-data:/var/lib/postgresql/data
```

```yaml
# docker-compose.dev.yml (development overrides)
version: "3.8"
services:
  webapp:
    image: myapp:dev
    environment:
      - NODE_ENV=development
      - DEBUG=true
    volumes:
      # Mount source code for hot reload in dev
      - ./src:/app/src
    ports:
      - "3000:3000"   # Different port for dev

  database:
    # Use the official image in dev (no volume backup needed)
    ports:
      - "5432:5432"   # Expose database port in dev for direct access
```

In Portainer, deploy with the override file by setting the compose file path to include both files.

## Method 3: Git Branch Deployments

Use different Git branches for different environments:

```
main branch     → Deploy to production
staging branch  → Deploy to staging
develop branch  → Deploy to development
```

In Portainer, create three stacks pointing to the same repository but different branches:

- **webapp-production**: `main` branch, production environment variables
- **webapp-staging**: `staging` branch, staging environment variables
- **webapp-dev**: `develop` branch, development environment variables

## Method 4: Portainer Stack Environment Files

Create environment-specific `.env` files in your repository:

```
.env.production:
  APP_VERSION=1.2.3
  LOG_LEVEL=warn
  REPLICA_COUNT=3

.env.staging:
  APP_VERSION=1.3.0-beta
  LOG_LEVEL=debug
  REPLICA_COUNT=1
```

In Portainer's Git stack configuration, specify which `.env` file to use per environment.

## Shared Configuration Management

For configurations shared across environments (like service names), use a config service:

```yaml
services:
  # Config server provides environment-specific configuration
  config:
    image: config-server:1.0
    volumes:
      - ./config:/config:ro
    environment:
      - ENV=${APP_ENVIRONMENT:-production}
    ports:
      - "8888:8888"

  webapp:
    image: myapp:1.2.3
    environment:
      # App fetches its config from the config server on startup
      - CONFIG_SERVER_URL=http://config:8888
      - APP_ENVIRONMENT=${APP_ENVIRONMENT:-production}
    depends_on:
      - config
```

## Summary

Environment-specific stack configurations in Portainer work best through environment variables for simple overrides, Git branches for significant configuration differences, and Docker Compose override files for development-specific behaviors like code mounting and port exposure. The key principle is maintaining a single source of truth and overriding only what differs per environment.
