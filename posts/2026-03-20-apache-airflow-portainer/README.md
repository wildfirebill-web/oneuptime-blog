# How to Deploy Apache Airflow via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Apache Airflow, Workflow Orchestration, Self-Hosted, Data Engineering

Description: Deploy Apache Airflow, the workflow orchestration platform, as a Docker Compose stack through Portainer for managing and scheduling data pipelines.

## Introduction

Apache Airflow is the industry-standard platform for authoring, scheduling, and monitoring data pipelines as DAGs (Directed Acyclic Graphs). The official Docker Compose setup makes it deployable via Portainer, giving your data engineering team a self-hosted orchestration platform.

## Prerequisites

- Portainer CE or BE installed
- Host with at least 4 GB RAM (8 GB recommended)
- Docker Engine 20.10+
- Docker Compose v2

## Step 1: Prepare the Environment

```bash
# Create directories for Airflow
mkdir -p /opt/airflow/{dags,logs,plugins,config}

# Set AIRFLOW_UID (required by official image)
echo -e "AIRFLOW_UID=$(id -u)" > /opt/airflow/.env

# Set correct ownership
chown -R $(id -u):0 /opt/airflow/{dags,logs,plugins,config}
```

## Step 2: Create the Docker Compose Stack

Navigate to **Stacks** → **Add Stack** → **Web Editor** in Portainer:

```yaml
version: "3.8"

x-airflow-common: &airflow-common
  image: apache/airflow:2.9.3
  environment: &airflow-common-env
    AIRFLOW__CORE__EXECUTOR: CeleryExecutor
    AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@airflow-db/airflow
    AIRFLOW__CELERY__RESULT_BACKEND: db+postgresql://airflow:airflow@airflow-db/airflow
    AIRFLOW__CELERY__BROKER_URL: redis://:@airflow-redis:6379/0
    AIRFLOW__CORE__FERNET_KEY: ""
    AIRFLOW__CORE__DAGS_ARE_PAUSED_AT_CREATION: "true"
    AIRFLOW__CORE__LOAD_EXAMPLES: "false"
    AIRFLOW__API__AUTH_BACKENDS: "airflow.api.auth.backend.basic_auth"
    AIRFLOW__WEBSERVER__SECRET_KEY: "your_secret_key_here"
    _PIP_ADDITIONAL_REQUIREMENTS: ""
  volumes:
    - /opt/airflow/dags:/opt/airflow/dags
    - /opt/airflow/logs:/opt/airflow/logs
    - /opt/airflow/plugins:/opt/airflow/plugins
    - /opt/airflow/config:/opt/airflow/config
  user: "${AIRFLOW_UID:-50000}:0"
  depends_on: &airflow-common-depends-on
    airflow-redis:
      condition: service_healthy
    airflow-db:
      condition: service_healthy
  networks:
    - airflow-net

services:
  # PostgreSQL — Airflow metadata database
  airflow-db:
    image: postgres:16
    container_name: airflow-db
    restart: unless-stopped
    environment:
      POSTGRES_USER: airflow
      POSTGRES_PASSWORD: airflow
      POSTGRES_DB: airflow
    volumes:
      - airflow_db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "airflow"]
      interval: 10s
      retries: 5
    networks:
      - airflow-net

  # Redis — Celery message broker
  airflow-redis:
    image: redis:7.2-bookworm
    container_name: airflow-redis
    restart: unless-stopped
    expose:
      - 6379
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 30s
      retries: 50
    networks:
      - airflow-net

  # Airflow Webserver — UI
  airflow-webserver:
    <<: *airflow-common
    container_name: airflow-webserver
    command: webserver
    ports:
      - "8080:8080"
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "--fail", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 5

  # Airflow Scheduler — triggers DAG runs
  airflow-scheduler:
    <<: *airflow-common
    container_name: airflow-scheduler
    command: scheduler
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "--fail", "http://localhost:8974/health"]
      interval: 30s
      timeout: 10s
      retries: 5

  # Airflow Worker — executes tasks
  airflow-worker:
    <<: *airflow-common
    container_name: airflow-worker
    command: celery worker
    restart: unless-stopped
    environment:
      <<: *airflow-common-env
      DUMB_INIT_SETSID: "0"
    healthcheck:
      test: ["CMD-SHELL", 'celery --app airflow.providers.celery.executors.celery_executor.app inspect ping -d "celery@$${HOSTNAME}"']
      interval: 30s
      timeout: 10s
      retries: 5

  # Airflow Init — initializes the database
  airflow-init:
    <<: *airflow-common
    container_name: airflow-init
    entrypoint: /bin/bash
    command:
      - -c
      - |
        airflow db migrate
        airflow users create \
          --username admin \
          --firstname Admin \
          --lastname User \
          --role Admin \
          --email admin@example.com \
          --password admin
    restart: on-failure

volumes:
  airflow_db_data:

networks:
  airflow-net:
    driver: bridge
```

## Step 3: Deploy the Stack

1. Name the stack `airflow`
2. Click **Deploy the stack**
3. Wait 2-3 minutes for the init container to complete

## Step 4: Access the Airflow UI

1. Open `http://your-host:8080`
2. Log in with `admin` / `admin`
3. **Change the admin password immediately**

## Step 5: Create Your First DAG

Create a file at `/opt/airflow/dags/hello_world.py`:

```python
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator

# Define default arguments
default_args = {
    'owner': 'airflow',
    'depends_on_past': False,
    'retries': 1,
    'retry_delay': timedelta(minutes=5),
}

# Define the DAG
with DAG(
    'hello_world',
    default_args=default_args,
    description='A simple hello world DAG',
    schedule=timedelta(days=1),  # Run daily
    start_date=datetime(2024, 1, 1),
    catchup=False,
    tags=['example'],
) as dag:

    def say_hello():
        print("Hello from Airflow!")
        return "Hello World"

    task = PythonOperator(
        task_id='say_hello',
        python_callable=say_hello,
    )
```

Airflow automatically detects DAGs in the `/dags` directory.

## Step 6: Scale Workers

To add more workers via Portainer:

1. Open the stack and click **Edit**
2. Increase the `airflow-worker` replica count
3. Or manually run: `docker compose up -d --scale airflow-worker=3`

## Conclusion

Apache Airflow deployed via Portainer gives your data team a powerful, self-hosted workflow orchestration platform. The CeleryExecutor setup with Redis and PostgreSQL is production-ready, and Portainer makes it easy to monitor each service's health, view logs, and scale workers as your pipeline workload grows.
