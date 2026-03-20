# How to Deploy Apache Superset via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Apache Superset, Business Intelligence, Self-Hosted, Data Visualization

Description: Deploy Apache Superset, the modern data exploration and visualization platform, as a Docker stack through Portainer for self-hosted business intelligence.

## Introduction

Apache Superset is an open-source business intelligence and data visualization platform that connects to dozens of databases and provides a drag-and-drop chart builder, interactive dashboards, and a SQL editor. Deploying it via Portainer makes it easy to manage in your self-hosted infrastructure.

## Prerequisites

- Portainer CE or BE installed
- Host with at least 4 GB RAM
- Docker Engine 20.10+
- A database to connect to (PostgreSQL, MySQL, Snowflake, etc.)

## Step 1: Generate a Secret Key

```bash
# Generate a secure secret key for Superset
openssl rand -base64 42
```

## Step 2: Create the Stack in Portainer

Navigate to **Stacks** → **Add Stack** → **Web Editor**:

```yaml
version: "3.8"

services:
  # Redis — Superset's cache and async query backend
  superset-redis:
    image: redis:7.2-alpine
    container_name: superset-redis
    restart: unless-stopped
    volumes:
      - superset_redis_data:/data
    networks:
      - superset-net
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # PostgreSQL — Superset's metadata database
  superset-db:
    image: postgres:16-alpine
    container_name: superset-db
    restart: unless-stopped
    environment:
      POSTGRES_DB: superset
      POSTGRES_USER: superset
      POSTGRES_PASSWORD: supersetpassword
    volumes:
      - superset_db_data:/var/lib/postgresql/data
    networks:
      - superset-net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U superset -d superset"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Superset — main application
  superset-app:
    image: apache/superset:4.0.2
    container_name: superset-app
    restart: unless-stopped
    depends_on:
      superset-db:
        condition: service_healthy
      superset-redis:
        condition: service_healthy
    ports:
      - "8088:8088"
    environment:
      # Must be a long random string
      SECRET_KEY: "your_generated_secret_key_here"
      # Database connection for Superset metadata
      SQLALCHEMY_DATABASE_URI: postgresql+psycopg2://superset:supersetpassword@superset-db:5432/superset
      # Redis for caching
      REDIS_URL: redis://superset-redis:6379/0
      SUPERSET_LOAD_EXAMPLES: "no"
    volumes:
      - superset_home:/app/superset_home
    networks:
      - superset-net
    command: >
      sh -c "
        superset db upgrade &&
        superset fab create-admin \
          --username admin \
          --firstname Admin \
          --lastname User \
          --email admin@superset.com \
          --password admin &&
        superset init &&
        gunicorn --bind 0.0.0.0:8088 --workers 4 --timeout 120 'superset.app:create_app()'
      "

  # Celery worker for async queries
  superset-worker:
    image: apache/superset:4.0.2
    container_name: superset-worker
    restart: unless-stopped
    depends_on:
      - superset-db
      - superset-redis
    environment:
      SECRET_KEY: "your_generated_secret_key_here"
      SQLALCHEMY_DATABASE_URI: postgresql+psycopg2://superset:supersetpassword@superset-db:5432/superset
      REDIS_URL: redis://superset-redis:6379/0
    volumes:
      - superset_home:/app/superset_home
    networks:
      - superset-net
    command: celery --app=superset.tasks.celery_app:app worker --loglevel=info

volumes:
  superset_redis_data:
  superset_db_data:
  superset_home:

networks:
  superset-net:
    driver: bridge
```

## Step 3: Deploy the Stack

1. Name the stack `superset`
2. Click **Deploy the stack**
3. Wait 2-4 minutes for the initialization to complete

## Step 4: Access Superset

1. Open `http://your-host:8088`
2. Log in with `admin` / `admin`
3. **Immediately change the admin password** under the user icon → **Profile**

## Step 5: Connect a Database

1. Go to **Settings** → **Database Connections** → **+ Database**
2. Select your database type (PostgreSQL, MySQL, BigQuery, etc.)
3. For PostgreSQL:
   ```
   postgresql://username:password@your-db-host:5432/dbname
   ```
4. Click **Test Connection** → **Connect**

## Step 6: Create a Dataset

1. **Datasets** → **+ Dataset**
2. Select the database and schema
3. Choose a table or enter a custom SQL query
4. Click **Add**

## Step 7: Build a Chart

1. Go to **Charts** → **+ Chart**
2. Select your dataset
3. Choose a chart type (Bar, Line, Pie, Table, etc.)
4. Drag metrics and dimensions into the configuration
5. Click **Update Chart** → **Save**

## Step 8: Create a Dashboard

1. **Dashboards** → **+ Dashboard**
2. Drag and drop charts onto the canvas
3. Set the refresh interval for auto-updating dashboards
4. Click **Save**

## Step 9: Add Custom Database Drivers

For databases not included in the base image:

```bash
# Access the running container
docker exec -it superset-app pip install mysqlclient

# Or add to a custom Dockerfile
FROM apache/superset:4.0.2
USER root
RUN pip install mysqlclient cx_Oracle pydruid
USER superset
```

## Step 10: Enable Async Queries

Large queries should run asynchronously. In `superset_config.py`:

```python
FEATURE_FLAGS = {
    "GLOBAL_ASYNC_QUERIES": True,
}
GLOBAL_ASYNC_QUERIES_REDIS_CONFIG = {
    "port": 6379,
    "host": "superset-redis",
    "db": 0,
}
```

Mount this as a config file in your compose volume.

## Conclusion

Apache Superset deployed via Portainer provides your team with enterprise-grade business intelligence capabilities — SQL lab, interactive charts, and shareable dashboards — all self-hosted. Portainer makes it straightforward to monitor the Celery workers, view application logs, and update to new versions as they're released.
