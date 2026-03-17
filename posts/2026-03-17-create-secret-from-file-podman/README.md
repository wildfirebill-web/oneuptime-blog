# How to Create a Secret from a File in Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Secrets, Security, Files

Description: Learn how to create Podman secrets from files to securely manage passwords, certificates, and configuration data.

---

> Creating secrets from files is ideal for multi-line data like TLS certificates, configuration files, and SSH keys that are cumbersome to pass via standard input.

While piping a simple string works for passwords, many secrets are stored in files: TLS certificates, JSON credentials, SSH keys, and configuration files. Podman can read secret data directly from a file on disk, making it easy to manage complex secrets.

---

## Creating a Secret from a File

```bash
# Create a secret from a file
podman secret create my_cert /path/to/certificate.pem

# The file contents become the secret value
```

## Common File-Based Secrets

```bash
# TLS certificate
podman secret create tls_cert /etc/ssl/certs/app.crt

# TLS private key
podman secret create tls_key /etc/ssl/private/app.key

# JSON credentials file
podman secret create gcp_credentials /path/to/service-account.json

# SSH private key
podman secret create ssh_key ~/.ssh/deploy_key

# Configuration file with sensitive values
podman secret create app_config /path/to/config.yaml
```

## Creating and Using a Secret from a File

```bash
# Create a password file
echo -n "secure-database-password" > /tmp/db_password.txt

# Create the secret from the file
podman secret create db_password /tmp/db_password.txt

# Remove the plaintext file for security
rm /tmp/db_password.txt

# Use the secret in a container
podman run -d \
  --name postgres \
  --secret db_password \
  -e POSTGRES_PASSWORD_FILE=/run/secrets/db_password \
  postgres:15
```

## Multi-Line Secrets

```bash
# Create a config file with multiple sensitive values
cat > /tmp/app-secrets.env << 'EOF'
DB_HOST=postgres.internal
DB_PASSWORD=secret123
API_KEY=sk-abc-def-ghi
SMTP_PASSWORD=mail_pass_456
EOF

# Create the secret from the multi-line file
podman secret create app_env_secrets /tmp/app-secrets.env

# Clean up the plaintext file
rm /tmp/app-secrets.env

# Mount the secret in a container
podman run -d \
  --name my-app \
  --secret app_env_secrets \
  my-app:latest
```

## TLS Certificate Pair Example

```bash
# Create secrets for TLS certificate and key
podman secret create nginx_cert ./certs/server.crt
podman secret create nginx_key ./certs/server.key

# Use both secrets in an nginx container
podman run -d \
  --name nginx-tls \
  --secret nginx_cert \
  --secret nginx_key \
  -p 443:443 \
  nginx-tls:latest

# Inside the container, files are at:
# /run/secrets/nginx_cert
# /run/secrets/nginx_key
```

## Verifying the Secret

```bash
# List all secrets
podman secret ls

# Inspect the secret metadata (value is not shown)
podman secret inspect my_cert
```

## Summary

Creating Podman secrets from files is straightforward using `podman secret create <name> <filepath>`. This approach is ideal for multi-line data like TLS certificates, JSON credentials, and configuration files. Always remove plaintext source files after creating the secret, and use descriptive names to identify the purpose of each secret.
