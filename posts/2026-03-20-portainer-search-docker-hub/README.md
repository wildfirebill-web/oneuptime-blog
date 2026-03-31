# How to Search Docker Hub for Images in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Image, Docker Hub, DevOps

Description: Learn how to search Docker Hub for container images directly from Portainer to find official images and community images.

## Introduction

Finding the right Docker image is the first step in containerizing an application. Portainer provides a Docker Hub search interface so you can browse available images without leaving the management console. This guide covers searching, evaluating, and selecting the right image.

## Prerequisites

- Portainer installed with a connected Docker environment

## Step 1: Search for Images in Portainer

1. Navigate to **Images** in Portainer.
2. Look for a **Search** field or a **Search for an image on Docker Hub** section.
3. Enter your search term.
4. Click **Search**.

Results show:
- **Image name**: e.g., `nginx`, `postgres`, `redis`
- **Stars**: Popularity indicator
- **Official**: Whether it's an official Docker-maintained image
- **Description**: Brief description

## Step 2: Understanding Search Results

### Official Images

Official images are maintained by Docker and verified:

```text
✓ Official images:
  nginx          - Official Nginx
  postgres       - Official PostgreSQL
  redis          - Official Redis
  node           - Official Node.js
  python         - Official Python
  mysql          - Official MySQL
  mongodb        - Official MongoDB (mongo)
  ubuntu         - Official Ubuntu
  alpine         - Official Alpine Linux
```

Official images:
- Are security-scanned by Docker.
- Have well-maintained Dockerfiles.
- Come from the root namespace (no organization prefix).
- Are marked with a ✓ or "Official" badge.

### Verified Publisher Images

These are from companies verified by Docker:

```text
bitnami/nginx       - Bitnami's production-hardened Nginx
bitnami/postgresql  - Bitnami's PostgreSQL
grafana/grafana     - Official Grafana from Grafana Labs
elastic/elasticsearch - Official from Elastic
hashicorp/vault     - Official from HashiCorp
```

### Community Images

```text
myorg/myapp  - Organization/username prefixed
```

Evaluate community images carefully:
- Check the pull count and star count.
- Review the Dockerfile on Docker Hub.
- Check when it was last updated.
- Prefer images with active maintenance.

## Step 3: Search Docker Hub via CLI

```bash
# Search Docker Hub from command line:

docker search nginx

# Output:
NAME                    DESCRIPTION                                 STARS   OFFICIAL
nginx                   Official build of Nginx.                    19234   [OK]
bitnami/nginx           Bitnami nginx Docker Image                  178
jwilder/nginx-proxy     Automated nginx proxy for Docker           2078
nginx/nginx-ingress     NGINX and NGINX Plus Ingress Controllers    98

# Filter by official images only:
docker search --filter is-official=true nginx

# Filter by minimum stars:
docker search --filter stars=100 database

# Limit results:
docker search --limit 5 python
```

## Step 4: Evaluate Images Before Using

Before pulling an unfamiliar image, check:

### Criteria for Image Selection

```bash
Factor              | Indicator of Quality
--------------------|--------------------------------------------
Official badge      | Maintained by Docker or original vendor
Star count          | >1000 stars = widely trusted
Pull count          | High pull count = battle-tested
Last updated        | Should be < 3 months ago for active projects
Image size          | Smaller = fewer attack surfaces
Dockerfile access   | Can review the Dockerfile on Docker Hub
```

### Check the Image on Docker Hub

Visit `hub.docker.com/_/nginx` (official) or `hub.docker.com/r/bitnami/nginx` (verified publisher):

- **Tags tab**: Available versions and their sizes.
- **Dockerfile**: Review what's inside.
- **Tags by OS/Architecture**: For multi-arch support.

## Step 5: Choosing the Right Tags

After finding the right image, choose the appropriate tag:

```bash
# PostgreSQL tag options:
postgres:latest         → Latest version (changes over time - risky for prod)
postgres:15             → Major version (gets minor/patch updates)
postgres:15.5           → Specific version (stable, predictable)
postgres:15-alpine      → Alpine-based (smaller image)
postgres:15.5-alpine    → Specific version + Alpine

# Node.js tag options:
node:20                 → Node.js 20 (LTS)
node:20-slim            → Slim Debian (smaller)
node:20-alpine          → Alpine (smallest)
node:20.11.0-alpine     → Pinned exact version (most reproducible)

# Nginx tag options:
nginx:latest            → Latest version
nginx:1.25              → Stable release
nginx:1.25-alpine       → Alpine-based (smaller)
nginx:alpine            → Latest nginx on Alpine
```

## Step 6: Check Image Vulnerability Scan

For security-conscious environments, check for vulnerabilities:

```bash
# Use Docker Scout (Docker Desktop/Hub):
docker scout cves nginx:latest

# Or use Trivy (open source):
docker run --rm aquasec/trivy:latest image nginx:alpine

# Sample output:
nginx:alpine (alpine 3.19.1)
============================
Total: 0 (UNKNOWN: 0, LOW: 0, MEDIUM: 0, HIGH: 0, CRITICAL: 0)
```

## Step 7: Find Images for Specific Technologies

Quick reference for common services:

```bash
# Web servers:
docker search nginx
docker search apache     # httpd image

# Databases:
docker search postgres
docker search mysql
docker search mariadb
docker search mongodb
docker search redis
docker search elasticsearch

# Messaging:
docker search rabbitmq
docker search kafka      # confluentinc/cp-kafka
docker search mosquitto  # eclipse-mosquitto

# Monitoring:
docker search prometheus
docker search grafana

# CI/CD:
docker search jenkins
docker search gitlab
docker search gitea

# Programming languages:
docker search python
docker search node
docker search golang
docker search java
docker search ruby
```

## Step 8: Pin Versions in Production

After finding the right image and tag:

```yaml
# docker-compose.yml - always pin versions in production
services:
  web:
    # Bad: image: nginx:latest (changes without notice)
    # Good: pin to specific version
    image: nginx:1.25.4-alpine

  db:
    image: postgres:15.5-alpine

  cache:
    image: redis:7.2.4-alpine
```

## Conclusion

Searching Docker Hub from Portainer gives you quick access to the vast ecosystem of available container images. Prioritize official images and verified publisher images, check stars and pull counts, review Dockerfiles when possible, and always pin to specific versions in production. The extra effort of choosing the right base image pays dividends in security, stability, and maintainability.
