# How to Install and Configure NetBox for IPv4 Address Management

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: NetBox, IPAM, IPv4, Network Management, Docker, Python

Description: Install NetBox using Docker Compose and configure it for IPv4 address management including setting up your first site, prefix, and IP space.

NetBox is an open-source IP address management (IPAM) and data center infrastructure management (DCIM) tool. It provides a comprehensive platform for documenting and managing IPv4 address space.

## Prerequisites

```bash
# Docker and Docker Compose are required

docker --version   # 20.10+
docker compose version  # v2.0+

git --version
```

## Step 1: Clone NetBox Docker

```bash
# Clone the official NetBox Docker repository
git clone -b release https://github.com/netbox-community/netbox-docker.git
cd netbox-docker
```

## Step 2: Configure NetBox

```bash
# Copy the default configuration
cp env/netbox.env env/netbox.env.bak

# Edit the main environment file
nano env/netbox.env
```

Key settings to configure:

```bash
# env/netbox.env
SUPERUSER_NAME=admin
SUPERUSER_EMAIL=admin@example.com
SUPERUSER_PASSWORD=SecurePassword123!
SECRET_KEY=<generate-with: python3 -c "import secrets; print(secrets.token_hex(50))">
ALLOWED_HOSTS=localhost 192.168.1.100 netbox.example.com
```

## Step 3: Start NetBox

```bash
# Pull images and start all services
docker compose pull
docker compose up -d

# Verify all containers are running
docker compose ps

# Expected containers:
# netbox          (web application)
# netbox-worker   (background tasks)
# postgres        (database)
# redis           (caching)
# redis-cache     (query caching)
```

## Step 4: Access the Web Interface

```bash
# NetBox is available at http://localhost:8080
curl http://localhost:8080

# Or access from browser: http://<SERVER_IP>:8080
# Login with SUPERUSER_NAME/SUPERUSER_PASSWORD
```

## Step 5: Initial IPAM Configuration via CLI

```bash
# Create initial data using the NetBox management shell
docker compose exec netbox python3 manage.py shell << 'EOF'
from ipam.models import RIR, Aggregate, Prefix

# Create an RIR (Regional Internet Registry) for RFC 1918 private space
rir, _ = RIR.objects.get_or_create(name="RFC 1918", slug="rfc-1918", is_private=True)

# Create an aggregate for the 10.0.0.0/8 space
agg, _ = Aggregate.objects.get_or_create(
    prefix="10.0.0.0/8",
    rir=rir,
    description="Private IPv4 space"
)

print("Created RIR and Aggregate")
EOF
```

## Step 6: Verify via API

```bash
# NetBox provides a REST API for all operations
# Get the API token from the web interface: Admin → API Tokens

# List all prefixes
curl -H "Authorization: Token <YOUR_TOKEN>" \
  http://localhost:8080/api/ipam/prefixes/ | python3 -m json.tool

# Create a prefix via API
curl -X POST \
  -H "Authorization: Token <YOUR_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"prefix": "10.100.0.0/16", "description": "Production network"}' \
  http://localhost:8080/api/ipam/prefixes/
```

## Configuring Persistent Storage

```bash
# Data is stored in Docker volumes by default
# For production, mount to host directories
# Edit docker-compose.yml to bind mount postgres data directory
```

NetBox's combination of web UI, REST API, and GraphQL makes it suitable for both manual IPAM documentation and automated IP allocation workflows.
