# How to Install and Configure phpIPAM for IPv4 Address Management

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: phpIPAM, IPAM, IPv4, Network Management, Docker, PHP

Description: Install phpIPAM using Docker Compose and configure it as an IPv4 address management system for tracking subnets and IP allocations.

phpIPAM is an open-source web-based IP address management application written in PHP. It offers a clean UI, subnet scanning, VLAN management, and REST API for IPv4 and IPv6 tracking.

## Step 1: Deploy phpIPAM with Docker Compose

```yaml
# docker-compose.yml
version: "3.8"

services:
  phpipam-web:
    image: phpipam/phpipam-www:latest
    ports:
      - "80:80"
    environment:
      - TZ=UTC
      - IPAM_DATABASE_HOST=phpipam-db
      - IPAM_DATABASE_PASS=phpipam_pass
      - IPAM_DATABASE_NAME=phpipam
      - IPAM_DATABASE_USER=phpipam
      # Disable SSL requirement for internal use
      - IPAM_NO_ONLINE_UPDATES=0
    volumes:
      - phpipam-logo:/phpipam/css/images/logo
      - phpipam-ca:/usr/local/share/ca-certificates
    restart: unless-stopped
    depends_on:
      - phpipam-db

  phpipam-db:
    image: mariadb:latest
    environment:
      - MYSQL_ROOT_PASSWORD=root_password
      - MYSQL_USER=phpipam
      - MYSQL_PASSWORD=phpipam_pass
      - MYSQL_DATABASE=phpipam
    volumes:
      - phpipam-db:/var/lib/mysql
    restart: unless-stopped

volumes:
  phpipam-logo:
  phpipam-ca:
  phpipam-db:
```

```bash
docker compose up -d

# Wait for containers to be healthy
docker compose ps
```

## Step 2: Initial Setup

1. Open `http://localhost` in your browser
2. Click **Install new phpipam database**
3. Enter the MySQL credentials from your compose file
4. Set the admin password
5. Log in with `admin` / your chosen password

## Step 3: Configure IPAM via the Web Interface

1. Go to **Administration → IP Sections**
2. Create a section: "Corporate Network"
3. Add subnets under the section

## Step 4: Create Sections and Subnets via API

```bash
# First, get an API token
curl -X POST \
  -H "Content-Type: application/json" \
  -d '{"app_id": "myapp", "password": "adminpassword"}' \
  "http://localhost/api/user/"

# Store the token
TOKEN="your-api-token-here"

# Create a section
curl -X POST \
  -H "token: $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "Corporate", "description": "Corporate network sections"}' \
  "http://localhost/api/myapp/sections/"

# Create a subnet
curl -X POST \
  -H "token: $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "subnet": "10.100.1.0",
    "mask": "24",
    "sectionId": "1",
    "description": "Web tier servers"
  }' \
  "http://localhost/api/myapp/subnets/"
```

## Step 5: Enable Subnet Scanning

phpIPAM can automatically scan subnets for live hosts:

```bash
# In the web UI: Administration → Scan Agents → Local
# Or configure via docker environment variable
# IPAM_PWRESET=0
```

Enable scanning in the subnet settings:
1. Edit a subnet
2. Check **Enable subnet scanning**
3. Set scan interval

## Viewing Subnet Utilization

```bash
# Get subnet details including used IPs
curl -H "token: $TOKEN" \
  "http://localhost/api/myapp/subnets/search/10.100.1.0/24/" \
  | python3 -m json.tool | grep -E "\"used\"|\"available\"|\"free\""
```

## Backing Up phpIPAM

```bash
# Database backup
docker compose exec phpipam-db mysqldump \
  -u phpipam -p phpipam_pass phpipam > phpipam-backup.sql

# Restore from backup
docker compose exec -T phpipam-db mysql \
  -u phpipam -p phpipam_pass phpipam < phpipam-backup.sql
```

phpIPAM provides a lighter-weight alternative to NetBox for organizations primarily needing subnet and IP tracking without full DCIM functionality.
