# How to Add Labels to Containers in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Containers, Organizations, DevOps

Description: Learn how to add and use Docker container labels in Portainer for organization, service discovery, and integration with tools like Traefik and Prometheus.

## Introduction

Docker labels are key-value metadata attached to containers, images, volumes, and networks. They're used for organizing resources, enabling service discovery (e.g., Traefik reverse proxy), configuring monitoring scraping (e.g., Prometheus), and applying deployment policies. Portainer makes it easy to add labels through the web UI.

## Prerequisites

- Portainer installed with a connected Docker environment

## Step 1: Add Labels During Container Creation

1. Navigate to **Containers > Add container**.
2. Scroll to the **Labels** tab.
3. Click **+ add label** for each label.
4. Enter the **Name** (key) and **Value**.

```text
# Organization labels

Name:   com.example.environment
Value:  production

Name:   com.example.team
Value:  platform-engineering

Name:   com.example.version
Value:  2.1.0

Name:   com.example.maintainer
Value:  john.doe@example.com
```

## Step 2: Common Label Conventions

Docker recommends using reverse-DNS notation for label keys:

```text
# Reverse DNS format (recommended)
com.example.environment=production
com.example.project=myapp
com.example.component=web

# Simple format (also common)
environment=production
app=myapp
tier=frontend
```

## Step 3: Traefik Integration via Labels

Traefik, a popular reverse proxy, uses Docker labels for automatic routing configuration:

```yaml
# docker-compose.yml with Traefik labels
version: "3.8"

services:
  web:
    image: myorg/webapp:latest
    restart: unless-stopped
    labels:
      # Enable Traefik for this container
      - "traefik.enable=true"
      # Define the router - matches requests to example.com
      - "traefik.http.routers.web.rule=Host(`example.com`)"
      # Use the HTTPS entrypoint
      - "traefik.http.routers.web.entrypoints=websecure"
      # Enable TLS/HTTPS
      - "traefik.http.routers.web.tls=true"
      # Use Let's Encrypt for certificates
      - "traefik.http.routers.web.tls.certresolver=letsencrypt"
      # Tell Traefik which port the container serves
      - "traefik.http.services.web.loadbalancer.server.port=8080"
```

In Portainer's label form, add each of these as a label row.

## Step 4: Prometheus Service Discovery via Labels

Prometheus can auto-discover containers to scrape using Docker labels:

```yaml
# Labels for Prometheus scraping
services:
  app:
    image: myorg/myapp:latest
    labels:
      # Tell Prometheus to scrape this container
      - "prometheus.scrape=true"
      # Metrics endpoint path
      - "prometheus.path=/metrics"
      # Metrics port
      - "prometheus.port=9090"
```

Prometheus `prometheus.yml` with Docker service discovery:

```yaml
scrape_configs:
  - job_name: 'docker-containers'
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
    relabel_configs:
      # Only scrape containers with prometheus.scrape=true
      - source_labels: [__meta_docker_container_label_prometheus_scrape]
        action: keep
        regex: "true"
      # Use the container's prometheus.path label
      - source_labels: [__meta_docker_container_label_prometheus_path]
        action: replace
        target_label: __metrics_path__
```

## Step 5: Labels for Watchtower Automation

Watchtower uses labels to control automatic container updates:

```yaml
labels:
  # Enable automatic updates for this container
  - "com.centurylinklabs.watchtower.enable=true"
  # Or disable updates for this container
  - "com.centurylinklabs.watchtower.enable=false"
  # Custom image pull credentials
  - "com.centurylinklabs.watchtower.scope=production"
```

## Step 6: Viewing Container Labels

After creating a container, view its labels:

1. Navigate to **Containers**.
2. Click the container name.
3. Go to the **Inspect** tab.
4. Find the `Labels` section in the JSON.

```bash
# Equivalent Docker CLI command:
docker inspect my-container --format '{{json .Config.Labels}}' | jq .

# Output:
{
  "com.example.environment": "production",
  "com.example.version": "2.1.0",
  "traefik.enable": "true",
  "traefik.http.routers.web.rule": "Host(`example.com`)"
}
```

## Step 7: Filter Containers by Labels

In Portainer, use the search/filter feature in the Containers list to find containers by label values. Via CLI:

```bash
# Find all containers with a specific label
docker ps --filter "label=com.example.environment=production"

# Find all containers with a label key (any value)
docker ps --filter "label=traefik.enable"
```

## Label Best Practices

```yaml
# Complete labeling example for a production service
labels:
  # Service identity
  - "com.example.app=myservice"
  - "com.example.version=2.1.0"
  - "com.example.environment=production"
  - "com.example.team=platform"

  # Operational metadata
  - "com.example.owner=john.doe@example.com"
  - "com.example.repo=github.com/myorg/myservice"
  - "com.example.created=2026-03-20"

  # Tool integration
  - "traefik.enable=true"
  - "prometheus.scrape=true"
```

- **Use a consistent prefix** for your organization's labels.
- **Include version information** to track what's deployed.
- **Add owner/contact info** for large teams.
- **Don't store sensitive data** in labels (they're visible in `docker inspect`).

## Conclusion

Labels in Docker and Portainer are a lightweight but powerful metadata system. By adding consistent labels to your containers, you enable automatic integration with tools like Traefik and Prometheus, make it easier to filter and find containers in large deployments, and maintain an audit trail of what's running and who's responsible for it.
