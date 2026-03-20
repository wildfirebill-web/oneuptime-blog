# How to Set Up a Python Development Environment with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Python, Development, DevOps, Home Lab

Description: Build a complete Python development environment with virtual environments, debugging, and hot-reload support using Docker and Portainer.

## Introduction

Containerized development environments ensure every team member uses the same Python version, dependencies, and tools. Using Docker with Portainer, you can spin up isolated Python development containers without polluting your host system. This guide covers building a complete Python dev environment with VS Code support.

## Step 1: Create a Python Development Dockerfile

```dockerfile
# Dockerfile.dev - Python development environment
FROM python:3.12-slim

# Set working directory
WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    git \
    curl \
    make \
    build-essential \
    # For psycopg2
    libpq-dev \
    # For PIL/Pillow
    libjpeg-dev \
    zlib1g-dev \
    # Cleanup
    && rm -rf /var/lib/apt/lists/*

# Install Python dev tools
RUN pip install --no-cache-dir \
    # Development tools
    black \
    isort \
    pylint \
    mypy \
    flake8 \
    # Testing
    pytest \
    pytest-cov \
    pytest-asyncio \
    # Debugging
    debugpy \
    # API development
    fastapi \
    uvicorn[standard] \
    pydantic \
    # Database
    sqlalchemy \
    alembic \
    psycopg2-binary \
    # Utilities
    python-dotenv \
    httpx \
    rich

# Copy requirements separately to leverage layer caching
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Create non-root user
RUN useradd -m -u 1000 developer
USER developer

# Expose ports
EXPOSE 8000   # FastAPI/Django
EXPOSE 5678   # debugpy remote debugger
```

## Step 2: Deploy the Dev Stack in Portainer

```yaml
# docker-compose.yml - Python Development Stack
version: "3.8"

networks:
  python_dev:
    driver: bridge

volumes:
  python_venv:
  postgres_data:
  redis_data:

services:
  # Python application with hot-reload
  app:
    build:
      context: .
      dockerfile: Dockerfile.dev
    container_name: python_app
    restart: unless-stopped
    ports:
      - "8000:8000"   # Application
      - "5678:5678"   # Remote debugger (VS Code)
    environment:
      - ENV=development
      - DATABASE_URL=postgresql://devuser:devpassword@postgres:5432/devdb
      - REDIS_URL=redis://redis:6379/0
      - DEBUG=true
    volumes:
      # Mount source code for hot-reload
      - ./src:/app/src
      # Persistent virtual environment (faster rebuilds)
      - python_venv:/app/.venv
    command: >
      uvicorn src.main:app
      --host 0.0.0.0
      --port 8000
      --reload
      --reload-dir src
    networks:
      - python_dev
    depends_on:
      - postgres
      - redis

  # PostgreSQL for development
  postgres:
    image: postgres:15-alpine
    container_name: dev_postgres
    restart: unless-stopped
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_DB=devdb
      - POSTGRES_USER=devuser
      - POSTGRES_PASSWORD=devpassword
    volumes:
      - postgres_data:/var/lib/postgresql/data
      # Auto-run init scripts
      - ./db/init:/docker-entrypoint-initdb.d
    networks:
      - python_dev

  # Redis for caching/queues
  redis:
    image: redis:7-alpine
    container_name: dev_redis
    restart: unless-stopped
    ports:
      - "6379:6379"
    networks:
      - python_dev

  # pgAdmin for database management
  pgadmin:
    image: dpage/pgadmin4:latest
    container_name: dev_pgadmin
    restart: unless-stopped
    ports:
      - "5050:80"
    environment:
      - PGADMIN_DEFAULT_EMAIL=admin@dev.local
      - PGADMIN_DEFAULT_PASSWORD=admin
    networks:
      - python_dev

  # Celery worker for background tasks
  celery_worker:
    build:
      context: .
      dockerfile: Dockerfile.dev
    container_name: celery_worker
    restart: unless-stopped
    command: celery -A src.celery_app worker --loglevel=info --autoreload
    environment:
      - DATABASE_URL=postgresql://devuser:devpassword@postgres:5432/devdb
      - REDIS_URL=redis://redis:6379/0
    volumes:
      - ./src:/app/src
    networks:
      - python_dev
    depends_on:
      - redis
      - postgres

  # Celery Flower monitoring
  flower:
    build:
      context: .
      dockerfile: Dockerfile.dev
    container_name: celery_flower
    restart: unless-stopped
    command: celery -A src.celery_app flower --port=5555
    ports:
      - "5555:5555"
    environment:
      - REDIS_URL=redis://redis:6379/0
    networks:
      - python_dev
    depends_on:
      - redis
```

## Step 3: Configure VS Code Remote Debugging

```json
// .vscode/launch.json - Remote debugging configuration
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Python: Remote Attach",
      "type": "python",
      "request": "attach",
      "connect": {
        "host": "localhost",
        "port": 5678
      },
      "pathMappings": [
        {
          "localRoot": "${workspaceFolder}/src",
          "remoteRoot": "/app/src"
        }
      ]
    }
  ]
}
```

```python
# src/main.py - Enable debugpy
import debugpy

# Enable remote debugger
debugpy.listen(("0.0.0.0", 5678))
print("Debugger waiting for connection on port 5678...")
debugpy.wait_for_client()  # Pause until debugger connects (dev only)

from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def root():
    return {"message": "Hello, Developer!"}
```

## Step 4: Running Tests in Portainer

```bash
# Run pytest in the container
docker exec python_app pytest src/tests/ -v --cov=src --cov-report=html

# View coverage report
# Open coverage report in browser
docker exec python_app python -m http.server 8001 --directory htmlcov/
```

## Step 5: Database Migrations with Alembic

```bash
# Initialize Alembic
docker exec python_app alembic init migrations

# Create a migration
docker exec python_app alembic revision --autogenerate -m "create users table"

# Apply migrations
docker exec python_app alembic upgrade head

# Rollback last migration
docker exec python_app alembic downgrade -1
```

## Step 6: VS Code Dev Containers Integration

```json
// .devcontainer/devcontainer.json
{
  "name": "Python Development",
  "dockerComposeFile": "../docker-compose.yml",
  "service": "app",
  "workspaceFolder": "/app",
  "extensions": [
    "ms-python.python",
    "ms-python.black-formatter",
    "ms-python.pylint",
    "ms-python.mypy-type-checker",
    "ms-azuretools.vscode-docker"
  ],
  "settings": {
    "python.defaultInterpreterPath": "/usr/local/bin/python",
    "editor.formatOnSave": true,
    "[python]": {
      "editor.defaultFormatter": "ms-python.black-formatter"
    }
  },
  "forwardPorts": [8000, 5678, 5432, 6379]
}
```

## Conclusion

Your Python development environment is now fully containerized and managed through Portainer. The setup includes hot-reload for rapid iteration, remote debugging for VS Code, a PostgreSQL database, Redis cache, and Celery worker — everything you need for modern Python development. Portainer makes it easy to view logs, restart services, and monitor resource usage across all components of your dev stack.
