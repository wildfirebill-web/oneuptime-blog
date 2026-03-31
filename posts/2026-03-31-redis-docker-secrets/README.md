# How to Set Up Redis with Docker Secrets

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Docker, Security

Description: Learn how to use Docker secrets to securely pass Redis passwords and TLS certificates to containers without exposing sensitive values in environment variables or images.

---

Docker secrets provide a secure way to manage sensitive data like Redis passwords and TLS certificates. Unlike environment variables, secrets are stored in Docker's encrypted secrets store and mounted as files inside containers.

## Docker Secrets Overview

```text
Environment variables:  Visible in docker inspect, logs, process lists
Docker secrets:         Encrypted at rest, only accessible inside container
                        Mounted as files at /run/secrets/<secret-name>
                        Not visible in docker inspect output
```

## Creating Secrets (Docker Swarm)

Docker secrets require Swarm mode:

```bash
# Initialize Swarm if not already in swarm mode
docker swarm init

# Create Redis password secret
echo "my-strong-redis-password-here" | docker secret create redis_password -

# Create TLS certificate secrets
docker secret create redis_tls_cert /path/to/redis.crt
docker secret create redis_tls_key /path/to/redis.key
docker secret create redis_ca_cert /path/to/ca.crt

# List secrets
docker secret ls
```

## Using Secrets in Docker Compose (Swarm)

```yaml
# docker-compose.swarm.yml
version: "3.8"

services:
  redis:
    image: redis:7-alpine
    command: /bin/sh -c "redis-server --requirepass $$(cat /run/secrets/redis_password)"
    secrets:
      - redis_password
    deploy:
      replicas: 1
    networks:
      - redis-net

secrets:
  redis_password:
    external: true

networks:
  redis-net:
    driver: overlay
    encrypted: true
```

## Custom Entrypoint for Secret Injection

Use a script to read the secret file and configure Redis:

```bash
#!/bin/sh
# redis-entrypoint.sh
set -e

REDIS_PASSWORD_FILE="/run/secrets/redis_password"
CONFIG_FILE="/etc/redis/redis.conf"

# Generate config from template
cp /etc/redis/redis.conf.template "$CONFIG_FILE"

if [ -f "$REDIS_PASSWORD_FILE" ]; then
  PASSWORD=$(cat "$REDIS_PASSWORD_FILE")
  echo "requirepass $PASSWORD" >> "$CONFIG_FILE"
  echo "masterauth $PASSWORD" >> "$CONFIG_FILE"
fi

# Handle TLS certs if present
if [ -f "/run/secrets/redis_tls_cert" ]; then
  echo "tls-cert-file /run/secrets/redis_tls_cert" >> "$CONFIG_FILE"
  echo "tls-key-file /run/secrets/redis_tls_key" >> "$CONFIG_FILE"
  echo "tls-ca-cert-file /run/secrets/redis_ca_cert" >> "$CONFIG_FILE"
  echo "tls-port 6380" >> "$CONFIG_FILE"
fi

exec redis-server "$CONFIG_FILE"
```

```dockerfile
FROM redis:7-alpine
COPY redis.conf.template /etc/redis/redis.conf.template
COPY redis-entrypoint.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/redis-entrypoint.sh
ENTRYPOINT ["redis-entrypoint.sh"]
```

## Compose Secrets for Local Development

For Docker Compose (non-Swarm), use file-based secrets:

```yaml
# docker-compose.dev.yml
version: "3.8"

services:
  redis:
    image: redis:7-alpine
    command: /bin/sh -c "redis-server --requirepass $$(cat /run/secrets/redis_password)"
    secrets:
      - redis_password

secrets:
  redis_password:
    file: ./secrets/redis_password.txt  # never commit this file
```

```bash
# Create the secrets file (add to .gitignore)
mkdir -p secrets
echo "dev-only-password" > secrets/redis_password.txt
echo "secrets/" >> .gitignore
```

## Verifying Secret Injection

```bash
# Check secret is mounted (file exists but content is protected)
docker exec redis ls /run/secrets/

# Verify Redis is using the password
docker exec redis redis-cli -a "$(cat secrets/redis_password.txt)" PING

# Confirm password is not visible in environment
docker exec redis env | grep -i redis  # should show nothing sensitive
docker inspect redis --format='{{json .Config.Env}}'  # no passwords
```

## Summary

Docker secrets provide encrypted, file-based delivery of sensitive Redis configuration values like passwords and TLS certificates. Use a custom entrypoint script to read secret files and build a runtime configuration, keeping credentials out of images, environment variables, and inspect output. For local development, use file-based compose secrets pointed at gitignored files, and use Docker Swarm's encrypted secret store in production.
