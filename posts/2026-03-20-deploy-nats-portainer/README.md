# How to Deploy NATS via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, NATS, Messaging, Docker, Microservices

Description: Deploy NATS high-performance messaging system using Portainer for cloud-native microservices communication.

## Introduction

Deploy NATS high-performance messaging system using Portainer for cloud-native microservices communication. This guide provides step-by-step instructions for deploying and configuring this service in your containerized infrastructure.

## Prerequisites

- Portainer installed with Docker
- At least 2 GB RAM available
- Basic understanding of messaging/caching concepts

## Step 1: Create the Stack in Portainer

Navigate to **Stacks** > **Add Stack** and use the following configuration:

```yaml
# docker-compose.yml
version: "3.8"

services:
  # Main service
  service:
    image: service-image:latest
    container_name: service
    restart: always
    ports:
      - "service-port:service-port"
    volumes:
      - service-data:/data
    environment:
      - CONFIG_KEY=config-value
    healthcheck:
      test: ["CMD", "service-healthcheck"]
      interval: 30s
      timeout: 10s
      retries: 3
    logging:
      driver: json-file
      options:
        max-size: "100m"
        max-file: "3"
    networks:
      - service-net

  # Application using the service
  app:
    image: my-app:latest
    container_name: app
    restart: always
    depends_on:
      service:
        condition: service_healthy
    environment:
      - SERVICE_URL=service://service:port
    networks:
      - service-net

volumes:
  service-data:

networks:
  service-net:
    driver: bridge
```

## Step 2: Configure the Service

Create configuration files via Portainer's Configs section:

```yaml
# Service configuration
server:
  host: 0.0.0.0
  port: 6379
  
logging:
  level: INFO
  
persistence:
  enabled: true
  directory: /data
  
security:
  # Enable authentication in production
  authentication: true
  password: ${SERVICE_PASSWORD}
```

## Step 3: Test the Connection

After deployment, test from Portainer's container console:

```bash
# Access the application container
# Portainer > Containers > app > Console

# Test connection to service
curl http://service:port/health

# Or use service-specific CLI
service-cli ping
service-cli info

# View service logs
docker logs service --tail 50 -f
```

## Step 4: Production Configuration

For production deployments, enhance security and reliability:

```yaml
services:
  service:
    image: service-image:latest
    restart: always
    # Resource limits
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 2G
        reservations:
          cpus: '0.5'
          memory: 512M
    # TLS configuration
    environment:
      - TLS_ENABLED=true
      - TLS_CERT_FILE=/certs/service.crt
      - TLS_KEY_FILE=/certs/service.key
      - PASSWORD=${SERVICE_PASSWORD}
    secrets:
      - service-tls-cert
      - service-tls-key
    
secrets:
  service-tls-cert:
    external: true
  service-tls-key:
    external: true
```

## Step 5: Set Up Monitoring

Monitor service performance through Portainer:

```yaml
  # Prometheus exporter for metrics
  service-exporter:
    image: service/exporter:latest
    container_name: service-exporter
    restart: always
    environment:
      - SERVICE_ADDR=service:port
    ports:
      - "9999:9999"
    networks:
      - service-net
```

Configure Prometheus to scrape metrics:

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'service'
    static_configs:
      - targets: ['service-exporter:9999']
```

## Step 6: Configure Persistence and Backups

Set up data persistence and automated backups:

```bash
#!/bin/bash
# backup.sh
BACKUP_DIR="/backups/service"
DATE=$(date +%Y%m%d_%H%M%S)
mkdir -p $BACKUP_DIR

# Create backup
docker exec service service-cli dump /tmp/backup.rdb
docker cp service:/tmp/backup.rdb $BACKUP_DIR/backup-$DATE.rdb

# Retain 7 days of backups
find $BACKUP_DIR -name "*.rdb" -mtime +7 -delete

echo "Backup complete: $BACKUP_DIR/backup-$DATE.rdb"
```

## Step 7: Scale and High Availability

For high availability, use multiple instances:

```yaml
services:
  service:
    image: service-image:latest
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
```

## Client Application Integration

Example integration code:

```python
# Python client example
import service_client

client = service_client.connect(
    host='localhost',
    port=port,
    password='your-password'
)

# Test connection
client.ping()

# Use the service
result = client.execute("operation", "key", "value")
print(f"Result: {result}")
```

## Conclusion

Deploying NATS via Portainer provides a managed, production-ready service that integrates seamlessly with your containerized infrastructure. Portainer's stack management simplifies configuration, updates, and monitoring while the persistent volume configuration ensures your data survives container restarts. Following the production configuration recommendations ensures your deployment is secure and reliable.
