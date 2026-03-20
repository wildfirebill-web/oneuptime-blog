# How to Connect Application Containers to Databases in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker Networking, Database Connection, Service Discovery, Environment Variables

Description: Learn how to connect application containers to database containers in Portainer using shared networks, service names, and environment variables.

---

Connecting application containers to database containers in Portainer is done through Docker's built-in DNS service discovery. Containers on the same network resolve each other by service name.

## Same-Stack Connection (Recommended)

When both the app and database are in the same Portainer stack, they share a network automatically:

```yaml
version: "3.8"

services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: apppassword
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d appdb"]
      interval: 10s
      timeout: 5s
      retries: 5

  api:
    image: my-api:latest
    environment:
      DATABASE_URL: "postgresql://appuser:apppassword@postgres:5432/appdb"
    depends_on:
      postgres:
        condition: service_healthy

volumes:
  postgres_data:
```

The `api` service uses `postgres` as the hostname — this resolves to the database container's IP automatically via Docker DNS.

## Cross-Stack Connection

When the database is in a separate stack, use an external named network:

```yaml
# In the database stack
networks:
  shared_db_net:
    driver: bridge
    name: shared_db_net   # Fixed name for cross-stack reference

services:
  postgres:
    networks:
      - shared_db_net
```

```yaml
# In the application stack
networks:
  shared_db_net:
    external: true   # Reference the existing network

services:
  api:
    environment:
      DATABASE_URL: "postgresql://appuser:apppassword@postgres:5432/appdb"
    networks:
      - shared_db_net
```

## Connection String Patterns by Database

| Database   | Connection String Format |
|------------|--------------------------|
| PostgreSQL | `postgresql://user:pass@service-name:5432/db` |
| MySQL      | `mysql://user:pass@service-name:3306/db` |
| MongoDB    | `mongodb://user:pass@service-name:27017/db` |
| Redis      | `redis://:password@service-name:6379/0` |
| MariaDB    | `mysql://user:pass@service-name:3306/db` |

## Using Portainer Stack Environment Variables

Store credentials in Portainer's stack environment variables instead of hardcoding them:

```yaml
services:
  api:
    environment:
      DB_HOST: ${DB_HOST}
      DB_PORT: ${DB_PORT}
      DB_NAME: ${DB_NAME}
      DB_USER: ${DB_USER}
      DB_PASSWORD: ${DB_PASSWORD}
```

Set these in Portainer's **Stacks > [stack name] > Environment variables** panel. They are injected at container start time and do not appear in the compose file.

## Connection Retry Logic

Databases take time to become ready. Add retry logic to your application:

```python
import time
import psycopg2
from psycopg2 import OperationalError

def connect_with_retry(dsn, retries=10, delay=3):
    for attempt in range(retries):
        try:
            conn = psycopg2.connect(dsn)
            print("Connected to database")
            return conn
        except OperationalError as e:
            if attempt < retries - 1:
                print(f"Connection failed, retrying in {delay}s... ({e})")
                time.sleep(delay)
            else:
                raise

conn = connect_with_retry("postgresql://appuser:apppassword@postgres:5432/appdb")
```

## Troubleshooting Connection Failures

```bash
# 1. Check if the database container is running
docker ps | grep postgres

# 2. Test DNS resolution from the app container
docker exec -it $(docker ps -qf name=api) nslookup postgres

# 3. Test TCP connectivity to the database port
docker exec -it $(docker ps -qf name=api) nc -zv postgres 5432

# 4. Verify both containers are on the same network
docker network inspect $(docker network ls -qf name=app) | jq '.[].Containers | keys'
```

## Secrets Management

For production, avoid putting passwords in environment variables. Use Docker secrets instead:

```yaml
services:
  api:
    secrets:
      - db_password
    environment:
      DB_PASSWORD_FILE: /run/secrets/db_password

secrets:
  db_password:
    external: true
```

Create the secret with `docker secret create db_password -` and read the password from the file in your application at startup.
