# How to Deploy Trivy as an Image Scanner with Portainer - Image

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Trivy, Security, Vulnerability Scanning, DevSecOps

Description: Deploy Trivy in server mode via Portainer to provide a shared vulnerability scanning service for your container registry and CI/CD pipeline.

## Introduction

Running Trivy in client-server mode separates the vulnerability database from the scanner client, enabling faster scans (no database download per scan) and centralized scanning infrastructure. This guide covers deploying Trivy server via Portainer, setting up the database update schedule, integrating with a private registry, and querying the API.

## Step 1: Deploy Trivy Server Stack

```yaml
# docker-compose.yml - Trivy server deployment

version: "3.8"

services:
  trivy-server:
    image: aquasec/trivy:latest
    container_name: trivy_server
    restart: unless-stopped
    command:
      - "server"
      - "--listen"
      - "0.0.0.0:4954"
      - "--cache-dir"
      - "/cache"
      - "--db-repository"
      - "ghcr.io/aquasecurity/trivy-db"
    volumes:
      - trivy_cache:/cache
    ports:
      - "4954:4954"
    environment:
      # GitHub token to avoid rate limiting when downloading DB
      - GITHUB_TOKEN=${GITHUB_TOKEN}
    healthcheck:
      test: ["CMD", "trivy", "health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Auto-update vulnerability database daily
  trivy-db-updater:
    image: aquasec/trivy:latest
    container_name: trivy_db_updater
    restart: unless-stopped
    volumes:
      - trivy_cache:/cache
    entrypoint: ["/bin/sh", "-c"]
    command:
      - |
        while true; do
          echo "Updating Trivy vulnerability database..."
          trivy image --cache-dir /cache --download-db-only
          echo "Database updated at $(date). Next update in 24h."
          sleep 86400
        done

volumes:
  trivy_cache:
    driver: local
```

## Step 2: Scan Images Using the Trivy Server

```bash
# Scan using the remote server (no local DB download required)
trivy image \
  --server http://trivy-server:4954 \
  --severity HIGH,CRITICAL \
  nginx:alpine

# From any machine pointing to your Trivy server
trivy image \
  --server http://YOUR_SERVER_IP:4954 \
  myapp/api:latest

# Scan with JSON output for processing
trivy image \
  --server http://YOUR_SERVER_IP:4954 \
  --format json \
  --output scan-results.json \
  myapp/api:latest

# Parse results with jq
cat scan-results.json | jq '.Results[].Vulnerabilities[] | select(.Severity == "CRITICAL") | {ID: .VulnerabilityID, Package: .PkgName, Fixed: .FixedVersion}'
```

## Step 3: Scan a Private Registry

```yaml
# docker-compose.yml - Trivy with private registry access
version: "3.8"

services:
  trivy-server:
    image: aquasec/trivy:latest
    container_name: trivy_server
    restart: unless-stopped
    command: ["server", "--listen", "0.0.0.0:4954"]
    volumes:
      - trivy_cache:/cache
      # Mount Docker credentials for private registry access
      - /root/.docker/config.json:/root/.docker/config.json:ro
    ports:
      - "4954:4954"
    environment:
      # Registry credentials via environment
      - TRIVY_USERNAME=registry_user
      - TRIVY_PASSWORD=registry_password

volumes:
  trivy_cache:
```

```bash
# Scan image from private registry
trivy image \
  --server http://trivy-server:4954 \
  registry.internal.com/myapp/api:latest

# With explicit credentials
trivy image \
  --username registry_user \
  --password registry_password \
  registry.internal.com/myapp/api:latest
```

## Step 4: Integration with Portainer Webhook Pipeline

```bash
#!/bin/bash
# pre-deploy-scan.sh - Called before Portainer webhook

TRIVY_SERVER="http://trivy-server:4954"
IMAGE=$1
MAX_CRITICAL=0
MAX_HIGH=5

echo "Scanning image: $IMAGE"

# Get vulnerability counts
RESULTS=$(trivy image \
  --server "$TRIVY_SERVER" \
  --format json \
  --quiet \
  "$IMAGE")

CRITICAL=$(echo "$RESULTS" | jq '[.Results[].Vulnerabilities[]? | select(.Severity == "CRITICAL")] | length')
HIGH=$(echo "$RESULTS" | jq '[.Results[].Vulnerabilities[]? | select(.Severity == "HIGH")] | length')

echo "Critical: $CRITICAL, High: $HIGH"

if [ "$CRITICAL" -gt "$MAX_CRITICAL" ]; then
  echo "BLOCKED: $CRITICAL critical vulnerabilities exceed threshold ($MAX_CRITICAL)"
  exit 1
fi

if [ "$HIGH" -gt "$MAX_HIGH" ]; then
  echo "BLOCKED: $HIGH high vulnerabilities exceed threshold ($MAX_HIGH)"
  exit 1
fi

echo "Scan passed. Proceeding with deployment."
exit 0
```

## Step 5: Scan Reports Dashboard

```yaml
# Add a scan results dashboard using Grafana
version: "3.8"

services:
  trivy-server:
    image: aquasec/trivy:latest
    restart: unless-stopped
    command: ["server", "--listen", "0.0.0.0:4954"]
    volumes:
      - trivy_cache:/cache

  # Store scan results in PostgreSQL
  scan-db:
    image: postgres:15-alpine
    restart: unless-stopped
    environment:
      - POSTGRES_DB=trivy_results
      - POSTGRES_USER=trivy
      - POSTGRES_PASSWORD=scan_pass
    volumes:
      - scan_db_data:/var/lib/postgresql/data

  # Scan result importer
  scan-importer:
    image: aquasec/trivy:latest
    restart: unless-stopped
    volumes:
      - trivy_cache:/cache
      - ./scan-all.sh:/scan-all.sh
    entrypoint: ["/bin/sh", "/scan-all.sh"]
    environment:
      - TRIVY_SERVER=http://trivy-server:4954
      - DB_HOST=scan-db

volumes:
  trivy_cache:
  scan_db_data:
```

## Step 6: Monitor Trivy Server Health

```bash
# Check Trivy server health
curl http://YOUR_SERVER_IP:4954/healthz
# Returns: OK

# Check database version
trivy image \
  --server http://YOUR_SERVER_IP:4954 \
  --format json \
  alpine:latest 2>&1 | jq '.SchemaVersion, .CreatedAt'

# Monitor cache directory size
du -sh /var/lib/docker/volumes/trivy_cache/_data/

# Test scan latency (server mode is much faster than standalone)
time trivy image \
  --server http://YOUR_SERVER_IP:4954 \
  --quiet \
  nginx:alpine
# With server: ~2-5 seconds (DB already loaded)
# Without server: ~30-60 seconds (downloads DB)
```

## Conclusion

Trivy server mode transforms vulnerability scanning from a slow per-invocation process into a fast shared service. The database loads once, stays in memory, and all scan clients benefit from instant access. Deploying Trivy server via Portainer alongside your other infrastructure services gives your team a reliable scanning endpoint for CI/CD pipelines, pre-deployment checks, and scheduled audits. Combined with Portainer's webhook deployment triggers, you can create gates that block deployments of images with critical vulnerabilities.
