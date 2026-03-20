# How to Use a Secret as a File Mount in a Podman Container

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Secrets, File Mount, Security

Description: Learn how to mount Podman secrets as files inside containers for secure access to sensitive data.

---

> Mounting secrets as files is the default and most secure way to deliver sensitive data to containers in Podman, keeping values out of environment variables and process listings.

When you attach a secret to a container in Podman, it is mounted as a file under `/run/secrets/` by default. This approach is preferred over environment variables because file contents are not visible in process listings or container inspection output.

---

## Basic File Mount

```bash
# Create a secret

echo -n "my-secret-password" | podman secret create db_password -

# Mount the secret as a file in the container
podman run -d \
  --name my-app \
  --secret db_password \
  my-app:latest

# The secret is available at /run/secrets/db_password inside the container
```

## Verifying the File Mount

```bash
# Check that the secret file exists inside the container
podman exec my-app ls -la /run/secrets/

# Read the secret value inside the container
podman exec my-app cat /run/secrets/db_password
```

## Mounting to a Custom Path

```bash
# Mount the secret to a custom location
podman run -d \
  --name my-app \
  --secret db_password,target=/app/config/db_password \
  my-app:latest

# The secret is now at /app/config/db_password instead of /run/secrets/db_password
```

## Multiple Secret File Mounts

```bash
# Create multiple secrets
echo -n "db-pass-123" | podman secret create db_password -
echo -n "redis-auth-456" | podman secret create redis_auth -
echo -n "sk-api-key-789" | podman secret create api_key -

# Mount all secrets as files
podman run -d \
  --name multi-secret-app \
  --secret db_password \
  --secret redis_auth \
  --secret api_key \
  multi-secret-app:latest

# All secrets are available under /run/secrets/
# /run/secrets/db_password
# /run/secrets/redis_auth
# /run/secrets/api_key
```

## Using File-Mounted Secrets in Applications

```bash
# PostgreSQL reads password from a file
echo -n "postgres-pass" | podman secret create pg_pass -

podman run -d \
  --name postgres \
  --secret pg_pass \
  -e POSTGRES_PASSWORD_FILE=/run/secrets/pg_pass \
  postgres:15

# Application reading secrets from files (Python example)
# In your application code:
# with open('/run/secrets/db_password', 'r') as f:
#     password = f.read().strip()
```

## TLS Certificate Mounts

```bash
# Mount TLS certificate and key as secrets
podman secret create tls_cert ./certs/server.crt
podman secret create tls_key ./certs/server.key

podman run -d \
  --name nginx-tls \
  --secret tls_cert,target=/etc/nginx/ssl/server.crt \
  --secret tls_key,target=/etc/nginx/ssl/server.key \
  -p 443:443 \
  nginx-tls:latest
```

## Summary

Mounting secrets as files is the default and most secure method for delivering sensitive data to Podman containers. Secrets appear as files under `/run/secrets/` by default, but you can customize the mount path with the `target` option. This approach keeps sensitive data out of environment variables, process listings, and container inspection output, making it the recommended way to handle credentials, certificates, and other sensitive data.
