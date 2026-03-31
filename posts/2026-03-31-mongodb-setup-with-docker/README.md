# How to Set Up MongoDB with Docker

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Docker, Container, Operations, Development

Description: A complete guide to running MongoDB in Docker with authentication, persistent volumes, custom configuration, and Docker Compose for multi-container setups.

---

## Running MongoDB in Docker

Docker makes it easy to run MongoDB locally or in production without managing system packages. The official MongoDB Docker image is available on Docker Hub as `mongo`.

```mermaid
flowchart TD
    App[Application Container] -->|27017| MongoDB[MongoDB Container]
    MongoDB --> Volume[Named Volume: mongodb-data]
    MongoDB --> Config[mongod.conf mount]
    subgraph Docker Network
        App
        MongoDB
    end
```

## Quick Start

Run a basic MongoDB container without authentication (development only):

```bash
docker run -d \
  --name mongodb \
  -p 27017:27017 \
  mongo:7.0
```

Connect to it with mongosh:

```bash
docker exec -it mongodb mongosh
```

## Running with Authentication

Pass environment variables to set the root username and password. The `MONGO_INITDB_ROOT_USERNAME` and `MONGO_INITDB_ROOT_PASSWORD` variables create an admin user on first startup.

```bash
docker run -d \
  --name mongodb \
  -p 27017:27017 \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=secretpassword \
  -v mongodb-data:/data/db \
  mongo:7.0
```

Connect with credentials:

```bash
docker exec -it mongodb mongosh \
  "mongodb://admin:secretpassword@127.0.0.1:27017/?authSource=admin"
```

## Persistent Data with Volumes

Without a volume, all data is lost when the container is removed. Use a named volume or a bind mount to persist data.

Named volume (recommended for production-like environments):

```bash
docker volume create mongodb-data

docker run -d \
  --name mongodb \
  -p 27017:27017 \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=secretpassword \
  -v mongodb-data:/data/db \
  mongo:7.0
```

Bind mount (useful for inspecting data files directly):

```bash
mkdir -p /opt/mongodb/data

docker run -d \
  --name mongodb \
  -p 27017:27017 \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=secretpassword \
  -v /opt/mongodb/data:/data/db \
  mongo:7.0
```

## Custom mongod.conf

Create a custom configuration file on the host and mount it into the container.

Create `/opt/mongodb/mongod.conf`:

```yaml
net:
  port: 27017
  bindIp: 0.0.0.0

storage:
  dbPath: /data/db
  wiredTiger:
    engineConfig:
      cacheSizeGB: 1

operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 100

systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true
```

Run the container with the custom config:

```bash
docker run -d \
  --name mongodb \
  -p 27017:27017 \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=secretpassword \
  -v mongodb-data:/data/db \
  -v /opt/mongodb/mongod.conf:/etc/mongod.conf:ro \
  mongo:7.0 --config /etc/mongod.conf
```

## Initialization Scripts

Place `.js` or `.sh` scripts in `/docker-entrypoint-initdb.d/` to run them on first startup (before the container's first run). These scripts run in the context of `MONGO_INITDB_DATABASE`.

Create an init script at `/opt/mongodb/init/01-create-app-user.js`:

```javascript
db = db.getSiblingDB("myapp");

db.createUser({
  user: "appUser",
  pwd: "appPassword",
  roles: [{ role: "readWrite", db: "myapp" }],
});

db.createCollection("orders");
```

Mount the init directory:

```bash
docker run -d \
  --name mongodb \
  -p 27017:27017 \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=secretpassword \
  -e MONGO_INITDB_DATABASE=myapp \
  -v mongodb-data:/data/db \
  -v /opt/mongodb/init:/docker-entrypoint-initdb.d:ro \
  mongo:7.0
```

## Docker Compose Setup

Docker Compose is ideal for running MongoDB alongside an application.

Create `docker-compose.yml`:

```yaml
version: "3.8"

services:
  mongodb:
    image: mongo:7.0
    container_name: mongodb
    restart: unless-stopped
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: secretpassword
      MONGO_INITDB_DATABASE: myapp
    ports:
      - "27017:27017"
    volumes:
      - mongodb-data:/data/db
      - ./init:/docker-entrypoint-initdb.d:ro
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s

  app:
    image: myapp:latest
    container_name: myapp
    restart: unless-stopped
    environment:
      MONGODB_URI: mongodb://appUser:appPassword@mongodb:27017/myapp?authSource=myapp
    depends_on:
      mongodb:
        condition: service_healthy
    networks:
      - app-network

volumes:
  mongodb-data:

networks:
  app-network:
    driver: bridge
```

Start the stack:

```bash
docker compose up -d
```

Check logs:

```bash
docker compose logs mongodb -f
```

Stop the stack:

```bash
docker compose down
```

Remove the stack including volumes:

```bash
docker compose down -v
```

## Connecting from the Application Container

From within the same Docker network, connect using the service name as the hostname:

```bash
mongodb://appUser:appPassword@mongodb:27017/myapp?authSource=myapp
```

## Backup and Restore

Backup using `mongodump`:

```bash
docker exec mongodb mongodump \
  --uri "mongodb://admin:secretpassword@127.0.0.1:27017/?authSource=admin" \
  --out /data/backup

docker cp mongodb:/data/backup ./backup
```

Restore with `mongorestore`:

```bash
docker cp ./backup mongodb:/data/backup

docker exec mongodb mongorestore \
  --uri "mongodb://admin:secretpassword@127.0.0.1:27017/?authSource=admin" \
  /data/backup
```

## Best Practices

- Always use a named volume or bind mount; never rely on the container's writable layer for data.
- Set `MONGO_INITDB_ROOT_USERNAME` and `MONGO_INITDB_ROOT_PASSWORD` - never run without authentication in any environment other than isolated local development.
- Use the healthcheck in Docker Compose so dependent services wait for MongoDB to be ready.
- Pin the image to a specific minor version (e.g., `mongo:7.0.8`) in production, not `latest`.
- Set `cacheSizeGB` in your custom `mongod.conf` to prevent MongoDB from consuming too much of the container's memory.
- Never expose port 27017 publicly; use Docker networks to isolate MongoDB from the internet.

## Summary

Running MongoDB in Docker is straightforward with the official `mongo` image. Use environment variables to set the root credentials, named volumes to persist data, and Docker Compose to manage multi-container setups. Mount a custom `mongod.conf` for production-like configuration control, and use `/docker-entrypoint-initdb.d/` scripts to initialize databases and users on first startup.
