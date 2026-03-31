# How to Connect to MySQL Running in Docker from the Host

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Docker, Networking, Database Administration, DevOps

Description: Learn how to connect to a MySQL instance running inside a Docker container from the host machine using port binding, Docker networks, and GUI tools.

---

## Overview

When MySQL runs inside a Docker container, connecting to it from the host requires either mapping the container's port to a host port or using Docker's networking features. The most common approach is port binding (`-p 3306:3306`), which exposes the container's MySQL port on the host's `127.0.0.1`.

## Running MySQL with Port Binding

```bash
docker run --name mysql-server \
    -e MYSQL_ROOT_PASSWORD=rootsecret \
    -p 3306:3306 \
    -d mysql:8.0
```

The `-p 3306:3306` flag binds host port 3306 to container port 3306.

## Connecting from the Host Command Line

```bash
# Using mysql CLI on the host (must be installed)
mysql -h 127.0.0.1 -P 3306 -u root -p

# Specify database directly
mysql -h 127.0.0.1 -P 3306 -u root -p mydb
```

Use `127.0.0.1` instead of `localhost` because `localhost` triggers a Unix socket connection by default on Linux, which does not reach the Docker container.

## Connecting via docker exec (No Local Client Required)

If you do not have a MySQL client on the host:

```bash
# Connect using the mysql client inside the container
docker exec -it mysql-server mysql -u root -p

# Run a single query without interactive mode
docker exec mysql-server mysql -u root -prootsecret -e "SHOW DATABASES;"
```

## Changing the Host Binding Port

If port 3306 is already in use on the host, map to a different port:

```bash
docker run --name mysql-server \
    -e MYSQL_ROOT_PASSWORD=rootsecret \
    -p 13306:3306 \
    -d mysql:8.0
```

Connect on the new host port:

```bash
mysql -h 127.0.0.1 -P 13306 -u root -p
```

## Allowing External Connections

By default, Docker binds to `0.0.0.0`, making MySQL accessible on all host interfaces. To restrict to a specific interface:

```bash
docker run --name mysql-server \
    -e MYSQL_ROOT_PASSWORD=rootsecret \
    -p 127.0.0.1:3306:3306 \
    -d mysql:8.0
```

Now only local host connections are accepted; remote connections are blocked.

## Creating a User That Accepts Remote Connections

The default `root` user in the official MySQL image allows connection from any host, but for applications create a dedicated user:

```sql
CREATE USER 'appuser'@'%' IDENTIFIED BY 'appsecret';
GRANT SELECT, INSERT, UPDATE, DELETE ON mydb.* TO 'appuser'@'%';
FLUSH PRIVILEGES;
```

## Connecting with Docker Compose

In a Compose setup, app containers connect to MySQL using the service name as the hostname:

```yaml
services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: rootsecret
      MYSQL_DATABASE: mydb
    ports:
      - "3306:3306"

  app:
    build: .
    environment:
      DB_HOST: mysql
      DB_PORT: 3306
    depends_on:
      - mysql
```

From the host, you still connect via `127.0.0.1:3306`.

## Connecting with GUI Tools

**MySQL Workbench:**
- Host: `127.0.0.1`
- Port: `3306`
- User: `root`

**DBeaver / TablePlus:**
- Host: `127.0.0.1`
- Port: `3306`

## Troubleshooting Connection Refused

```bash
# Check if the container is running
docker ps

# Check container logs
docker logs mysql-server | tail -20

# Check port mapping
docker port mysql-server
```

## Summary

Connecting to MySQL in Docker from the host requires the container to publish port 3306 via `-p 3306:3306`. Use `127.0.0.1` (not `localhost`) from the host command line to avoid Unix socket routing. For GUI tools and application code, connect using `127.0.0.1` and the mapped host port. Restrict the binding to `127.0.0.1:3306:3306` to prevent unintended external access.
