# How to Run MongoDB in Docker with Persistent Volumes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Docker, Volume, Container, Persistence

Description: Learn how to run MongoDB in Docker with persistent volumes so data survives container restarts, including authentication, initialization scripts, and compose setup.

---

## Overview

Running MongoDB in Docker is convenient for development and testing, but without persistent volumes the data is lost when the container is removed. This guide shows how to configure volumes, enable authentication, and use Docker Compose for a production-ready local setup.

## Running MongoDB with a Named Volume

```bash
docker run -d \
  --name mongodb \
  -p 27017:27017 \
  -v mongodb_data:/data/db \
  mongo:7.0
```

The `-v mongodb_data:/data/db` flag creates a named volume that persists across container restarts and removals.

## Enabling Authentication

```bash
docker run -d \
  --name mongodb \
  -p 27017:27017 \
  -v mongodb_data:/data/db \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=securepassword \
  mongo:7.0
```

Connect with authentication:

```bash
mongosh "mongodb://admin:securepassword@localhost:27017"
```

## Using Docker Compose

A `docker-compose.yml` for a complete MongoDB setup with authentication and initialization:

```yaml
version: "3.8"

services:
  mongodb:
    image: mongo:7.0
    container_name: mongodb
    restart: unless-stopped
    ports:
      - "27017:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: securepassword
      MONGO_INITDB_DATABASE: mydb
    volumes:
      - mongodb_data:/data/db
      - ./init-scripts:/docker-entrypoint-initdb.d:ro

volumes:
  mongodb_data:
    driver: local
```

## Running Initialization Scripts

Place `.js` or `.sh` files in the `init-scripts` directory. They run once when the container is first created:

```javascript
// init-scripts/01-create-user.js
db = db.getSiblingDB("mydb");
db.createUser({
  user: "appuser",
  pwd: "apppassword",
  roles: [{ role: "readWrite", db: "mydb" }]
});
db.products.insertMany([
  { name: "Widget A", price: 9.99 },
  { name: "Widget B", price: 14.99 }
]);
```

## Using a Host Directory Instead of a Named Volume

For direct access to data files on disk:

```bash
docker run -d \
  --name mongodb \
  -p 27017:27017 \
  -v /var/data/mongodb:/data/db \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=securepassword \
  mongo:7.0
```

Ensure the host directory has the correct permissions:

```bash
sudo chown -R 999:999 /var/data/mongodb
```

## Backing Up MongoDB Data

```bash
# Run mongodump inside the container
docker exec mongodb mongodump \
  --username admin \
  --password securepassword \
  --out /tmp/backup

# Copy backup to host
docker cp mongodb:/tmp/backup ./mongodb_backup
```

## Summary

Run MongoDB in Docker with a named volume (`-v mongodb_data:/data/db`) to persist data across container restarts. Use environment variables to set the root credentials, and place initialization scripts in `/docker-entrypoint-initdb.d`. Docker Compose makes managing authentication, volumes, and initialization scripts straightforward for local and staging environments.
