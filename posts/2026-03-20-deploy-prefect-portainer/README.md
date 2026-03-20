# How to Deploy Prefect via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Prefect, Data Workflow, Docker, Python

Description: Deploy Prefect workflow orchestration server using Portainer to schedule and monitor Python data pipelines.

## Introduction

Prefect is a modern workflow orchestration framework for data pipelines. It provides a Python-first API to define flows (workflows) and tasks, plus a web UI for scheduling, monitoring, and managing runs. Prefect 2.x (Prefect Orion) supports self-hosted server deployments.

## Prerequisites

- Portainer installed with Docker
- Python 3.8+ for running local flows

## Step 1: Create the Stack in Portainer

Navigate to **Stacks** > **Add Stack**:

```yaml
# docker-compose.yml - Prefect Server
version: "3.8"

services:
  prefect-server:
    image: prefecthq/prefect:2.19.7-python3.12
    container_name: prefect_server
    restart: unless-stopped
    ports:
      - "4200:4200"
    volumes:
      - prefect_data:/root/.prefect
    environment:
      - PREFECT_UI_ENABLED=true
      - PREFECT_API_URL=http://0.0.0.0:4200/api
      - PREFECT_SERVER_API_HOST=0.0.0.0
      - PREFECT_UI_API_URL=http://${PREFECT_HOST}:4200/api
      - PREFECT_API_DATABASE_CONNECTION_URL=postgresql+asyncpg://prefect:${DB_PASSWORD}@prefect_postgres:5432/prefect
    command: prefect server start --host 0.0.0.0 --port 4200
    depends_on:
      prefect_postgres:
        condition: service_healthy
    networks:
      - prefect_net

  prefect_postgres:
    image: postgres:16-alpine
    container_name: prefect_postgres
    restart: unless-stopped
    volumes:
      - prefect_postgres_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=prefect
      - POSTGRES_USER=prefect
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U prefect"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - prefect_net

volumes:
  prefect_data:
  prefect_postgres_data:

networks:
  prefect_net:
    driver: bridge
```

## Step 2: Set Environment Variables in Portainer

```
DB_PASSWORD=your-postgres-password
PREFECT_HOST=prefect.yourdomain.com
```

## Step 3: Access the Prefect UI

Open `http://<host>:4200` to view the Prefect dashboard.

## Step 4: Deploy a Flow

Install the Prefect client and point it at your server:

```bash
pip install prefect

# Configure Prefect client to use your server
prefect config set PREFECT_API_URL=http://<host>:4200/api
```

Create a flow:

```python
# my_flow.py
from prefect import flow, task
from datetime import timedelta

@task(retries=3, retry_delay_seconds=10)
def fetch_data(url: str) -> dict:
    import requests
    response = requests.get(url)
    return response.json()

@task
def process_data(data: dict) -> None:
    print(f"Processing {len(data)} records")

@flow(name="data-pipeline", log_prints=True)
def data_pipeline(url: str = "https://api.example.com/data"):
    data = fetch_data(url)
    process_data(data)

if __name__ == "__main__":
    data_pipeline()
```

## Step 5: Schedule and Deploy the Flow

```bash
# Create a deployment with a schedule
prefect deployment build my_flow.py:data_pipeline \
  --name "production" \
  --cron "0 * * * *" \
  --apply

# Run the flow manually
prefect deployment run "data-pipeline/production"
```

## Step 6: Start a Worker to Execute Flows

```bash
# Workers pull flow runs from the server and execute them
prefect worker start --pool default-agent-pool
```

## Conclusion

Prefect Server requires PostgreSQL for storing flow metadata, run history, and schedules. The `PREFECT_UI_API_URL` must be reachable from the browser — set it to the external hostname of your server. Workers (agents) poll the server for scheduled runs and execute them locally or in a Docker container. Use `@task(cache_key_fn=task_input_hash)` for idempotent pipelines that skip recomputation of unchanged inputs.
