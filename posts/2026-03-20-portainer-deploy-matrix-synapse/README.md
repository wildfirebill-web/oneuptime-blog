# How to Deploy Matrix/Synapse via Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Matrix, Synapse, Decentralized Chat, Self-Hosted, Federation

Description: Deploy Matrix Synapse homeserver via Portainer for decentralized, end-to-end encrypted team communication with federation support and Element web client.

## Introduction

Matrix is an open standard for decentralized, real-time communication with end-to-end encryption. Synapse is the reference Matrix homeserver implementation. By hosting your own Synapse server, you have complete control over your communications and can federate with the wider Matrix network.

## Prerequisites

- A domain name (matrix.example.com)
- DNS configured
- Port 8448 accessible for federation (optional but recommended)

## Step 1: Generate Synapse Configuration

```bash
# Generate initial Synapse configuration

docker run --rm \
  -v $(pwd)/synapse-config:/data \
  -e SYNAPSE_SERVER_NAME=example.com \
  -e SYNAPSE_REPORT_STATS=no \
  matrixdotorg/synapse:latest generate

# This creates homeserver.yaml in ./synapse-config/
```

## Deploy as a Stack

```yaml
version: "3.8"

services:
  synapse:
    image: matrixdotorg/synapse:latest
    container_name: synapse
    environment:
      - SYNAPSE_SERVER_NAME=example.com
      - SYNAPSE_REPORT_STATS=no
    volumes:
      - ./synapse-config:/data
    ports:
      - "8008:8008"    # Client-Server API
      - "8448:8448"    # Server-Server Federation
    depends_on:
      - synapse-db
    restart: unless-stopped

  synapse-db:
    image: postgres:16-alpine
    container_name: synapse-db
    environment:
      POSTGRES_DB: synapse
      POSTGRES_USER: synapse
      POSTGRES_PASSWORD: synapse_db_password
      POSTGRES_INITDB_ARGS: "--encoding=UTF-8 --lc-collate=C --lc-ctype=C"
    volumes:
      - synapse_db:/var/lib/postgresql/data
    restart: unless-stopped

  # Element Web - Matrix web client
  element:
    image: vectorim/element-web:latest
    container_name: element
    volumes:
      - ./element-config.json:/app/config.json:ro
    ports:
      - "8080:80"
    restart: unless-stopped

volumes:
  synapse_db:
```

## Configure Synapse for PostgreSQL

Edit `synapse-config/homeserver.yaml`:

```yaml
# Database configuration (replace SQLite with PostgreSQL)
database:
  name: psycopg2
  args:
    user: synapse
    password: synapse_db_password
    database: synapse
    host: synapse-db
    cp_min: 5
    cp_max: 10

# Registration
enable_registration: false   # Disable public registration
registration_requires_token: true

# Turn server for VoIP
turn_uris:
  - "turn:turn.example.com?transport=udp"
  - "turn:turn.example.com?transport=tcp"
```

## Element Web Configuration

Create `element-config.json`:

```json
{
  "default_server_config": {
    "m.homeserver": {
      "base_url": "https://matrix.example.com",
      "server_name": "example.com"
    }
  },
  "disable_custom_urls": false,
  "disable_guests": true,
  "default_theme": "dark",
  "features": {
    "feature_video_rooms": true
  }
}
```

## Create Admin User

```bash
# Register an admin user
docker exec -it synapse register_new_matrix_user \
  -c /data/homeserver.yaml \
  -u admin \
  -p admin_password \
  -a \    # -a = admin
  http://localhost:8008
```

## Conclusion

Matrix/Synapse deployed via Portainer provides a decentralized, federated communication platform with end-to-end encryption. Unlike centralized platforms, Matrix gives you data ownership while allowing communication with users on other Matrix servers. Element Web provides a polished client interface, and the federation capability means your team can communicate with the wider Matrix ecosystem.
