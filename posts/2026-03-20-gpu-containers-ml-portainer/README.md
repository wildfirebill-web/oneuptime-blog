# How to Set Up GPU Containers for ML Workloads in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, GPU, Machine Learning, NVIDIA, Docker

Description: Configure NVIDIA GPU support in Docker containers managed by Portainer for accelerated machine learning workloads.

## Introduction

How to Set Up GPU Containers for ML Workloads in Portainer provides a comprehensive guide to deploying and configuring this technology in a containerized environment managed by Portainer. Whether you're setting up a development environment or production workload, this guide covers everything you need.

## Prerequisites

- Portainer installed with Docker
- Adequate hardware resources (CPU/GPU/RAM as needed)
- Docker and Docker Compose installed
- Network access to required services

## Step 1: Prepare Your Environment

Ensure your system meets the requirements:

```bash
# Check available resources

free -h
nproc
df -h

# For GPU workloads, check NVIDIA availability
nvidia-smi
docker run --rm --gpus all nvidia/cuda:12.0-base-ubuntu22.04 nvidia-smi
```

## Step 2: Create the Stack in Portainer

Navigate to **Stacks** > **Add Stack** and use the following docker-compose.yml:

```yaml
version: "3.8"

services:
  app:
    image: relevant-image:latest
    container_name: ml-app
    restart: always
    ports:
      - "8080:8080"
    volumes:
      - app-data:/data
      - ./models:/models
    environment:
      - ENV=production
      - LOG_LEVEL=info
    # GPU support (uncomment if needed)
    # deploy:
    #   resources:
    #     reservations:
    #       devices:
    #         - driver: nvidia
    #           count: all
    #           capabilities: [gpu]
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    logging:
      driver: json-file
      options:
        max-size: "100m"
        max-file: "3"
    networks:
      - ml-net

volumes:
  app-data:

networks:
  ml-net:
    driver: bridge
```

## Step 3: Configure the Application

Access configuration through Portainer's Configs section:

```yaml
# Application configuration
server:
  host: 0.0.0.0
  port: 8080

storage:
  backend: local
  path: /data

logging:
  level: INFO
  format: json

# Database connection (if applicable)
database:
  host: postgres
  port: 5432
  name: appdb
```

## Step 4: Initialize and Verify

After deployment, verify the service is running:

```bash
# Check container health
docker ps | grep ml-app

# View logs via Portainer or CLI
docker logs ml-app --tail 50

# Test the API endpoint
curl http://localhost:8080/health

# Access UI at http://your-server:8080
```

## Step 5: Configure Persistent Storage

Ensure data persists across container restarts:

```yaml
# In docker-compose.yml
volumes:
  app-data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /data/ml-app
```

Create the host directory:

```bash
mkdir -p /data/ml-app
chmod 755 /data/ml-app
```

## Step 6: Monitor Performance

Use Portainer's built-in monitoring and set up Prometheus metrics:

```yaml
# Add Prometheus metrics scraping
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
```

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'ml-app'
    static_configs:
      - targets: ['ml-app:8080']
    metrics_path: /metrics
```

## Step 7: Backup and Recovery

Configure automated backups via Portainer:

```bash
#!/bin/bash
# backup.sh - run as a Portainer Edge Job or scheduled task

BACKUP_DIR="/backups/ml-app"
DATE=$(date +%Y%m%d_%H%M%S)
mkdir -p $BACKUP_DIR

# Backup application data
docker run --rm \
  -v app-data:/source:ro \
  -v $BACKUP_DIR:/backup \
  alpine tar czf /backup/app-data-$DATE.tar.gz -C /source .

echo "Backup completed: app-data-$DATE.tar.gz"

# Retain last 7 backups
ls -t $BACKUP_DIR/*.tar.gz | tail -n +8 | xargs rm -f
```

## Conclusion

How to Set Up GPU Containers for ML Workloads in Portainer using Portainer provides a streamlined approach to deploying and managing sophisticated workloads. Portainer's visual interface reduces operational complexity while its API enables automation and GitOps workflows. This deployment pattern scales from development environments to production clusters, making it suitable for teams at any stage of their containerization journey.
