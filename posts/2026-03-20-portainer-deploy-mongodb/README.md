# How to Deploy MongoDB via Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, MongoDB, NoSQL, Database, Self-Hosted

Description: Deploy MongoDB via Portainer with authentication, persistent storage, and Mongo Express for web-based document database management.

## Introduction

MongoDB is the leading NoSQL document database, popular for applications with flexible schemas, high read/write throughput, and horizontal scaling. Deploying it via Portainer provides persistent storage and a web admin interface through Mongo Express.

## Deploy as a Stack

In Portainer, create a stack named `mongodb`:

```yaml
version: "3.8"

services:
  mongodb:
    image: mongo:7.0
    container_name: mongodb
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: change_this_root_password
      MONGO_INITDB_DATABASE: myapp
    volumes:
      # Persistent data storage
      - mongodb_data:/data/db
      # MongoDB configuration
      - ./mongod.conf:/etc/mongod.conf:ro
      # Initialization scripts
      - ./init:/docker-entrypoint-initdb.d:ro
    ports:
      - "27017:27017"
    command: ["mongod", "--config", "/etc/mongod.conf"]
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  # Mongo Express - web-based MongoDB admin
  mongo-express:
    image: mongo-express:latest
    container_name: mongo-express
    environment:
      ME_CONFIG_MONGODB_ADMINUSERNAME: admin
      ME_CONFIG_MONGODB_ADMINPASSWORD: change_this_root_password
      ME_CONFIG_MONGODB_SERVER: mongodb
      ME_CONFIG_BASICAUTH_USERNAME: admin
      ME_CONFIG_BASICAUTH_PASSWORD: express_password
    ports:
      - "8081:8081"
    depends_on:
      mongodb:
        condition: service_healthy
    restart: unless-stopped

volumes:
  mongodb_data:
```

## MongoDB Configuration

Create `mongod.conf`:

```yaml
# mongod.conf

storage:
  dbPath: /data/db
  engine: wiredTiger
  wiredTiger:
    engineConfig:
      # Cache size: 50% of RAM (MongoDB default)
      cacheSizeGB: 1.0

systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log

net:
  port: 27017
  bindIp: 0.0.0.0

security:
  authorization: enabled

operationProfiling:
  # Log slow operations (>100ms)
  slowOpThresholdMs: 100
  mode: slowOp
```

## Initialization Script

Create `init/01-create-user.js`:

```javascript
// Create application user with limited permissions
db = db.getSiblingDB('myapp');

db.createUser({
  user: 'myapp_user',
  pwd: 'change_this_app_password',
  roles: [
    { role: 'readWrite', db: 'myapp' }
  ]
});

// Create indexes for common queries
db.users.createIndex({ email: 1 }, { unique: true });
db.users.createIndex({ createdAt: -1 });
db.sessions.createIndex({ expiresAt: 1 }, { expireAfterSeconds: 0 });

print('MongoDB initialization complete');
```

## Connecting Applications

```yaml
services:
  webapp:
    image: myapp:latest
    environment:
      # MongoDB connection string with authentication
      MONGODB_URI: mongodb://myapp_user:change_this_app_password@mongodb:27017/myapp
      # Or with admin credentials:
      MONGODB_URI_ADMIN: mongodb://admin:change_this_root_password@mongodb:27017/admin
```

## Backup and Restore

```bash
# Backup all databases

docker exec mongodb mongodump \
  -u admin -p "change_this_root_password" \
  --authenticationDatabase admin \
  --out /tmp/backup

# Copy backup from container
docker cp mongodb:/tmp/backup ./mongodb-backup-$(date +%Y%m%d)

# Restore
docker exec mongodb mongorestore \
  -u admin -p "change_this_root_password" \
  --authenticationDatabase admin \
  --dir /tmp/backup
```

## Conclusion

MongoDB deployed via Portainer provides a flexible document database with Mongo Express for visual data management. The persistent volume ensures data survives container updates, and authentication is enabled from the start for security. The initialization script creates application-specific users with appropriate permissions, following the principle of least privilege.
