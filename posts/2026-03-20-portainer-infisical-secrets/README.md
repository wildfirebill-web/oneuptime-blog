# How to Deploy Infisical Secrets Manager via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Infisical, Secrets, Security, Open Source

Description: Deploy Infisical open-source secrets manager via Portainer and use it to manage secrets for containerized applications.

## Introduction

Infisical is an open-source secrets manager that provides a GitHub-like UI for managing application secrets across environments. It supports automatic secret rotation, audit logs, and SDK/CLI integration. Deploying Infisical via Portainer gives your team a self-hosted alternative to HashiCorp Vault with a simpler setup.

## Deploying Infisical via Portainer

```yaml
# infisical-stack.yml
version: '3.8'

services:
  infisical:
    image: infisical/infisical:latest
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      ENCRYPTION_KEY: "${ENCRYPTION_KEY}"  # Generate with: openssl rand -hex 16
      AUTH_SECRET: "${AUTH_SECRET}"          # Generate with: openssl rand -base64 32
      REDIS_URL: "redis://redis:6379"
      MONGO_URL: "mongodb://mongo:27017/infisical"
      SITE_URL: "https://infisical.example.com"
      SMTP_HOST: "${SMTP_HOST}"
      SMTP_PORT: "587"
      SMTP_USERNAME: "${SMTP_USERNAME}"
      SMTP_PASSWORD: "${SMTP_PASSWORD}"
      SMTP_FROM_ADDRESS: "noreply@example.com"
    depends_on:
      - mongo
      - redis
    networks:
      - infisical-net

  mongo:
    image: mongo:6
    restart: unless-stopped
    volumes:
      - mongo-data:/data/db
    environment:
      MONGO_INITDB_DATABASE: infisical
    networks:
      - infisical-net

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    volumes:
      - redis-data:/data
    networks:
      - infisical-net

volumes:
  mongo-data:
  redis-data:

networks:
  infisical-net:
```

## Initial Setup

```bash
# Access Infisical at https://infisical.example.com
# 1. Create admin account
# 2. Create an organization
# 3. Create a project (e.g., "my-app")
# 4. Add secrets to each environment (development, staging, production)

# Install Infisical CLI
curl -1sLf \
  'https://dl.cloudsmith.io/public/infisical/infisical-cli/setup.alpine.sh' \
  | bash
apk add infisical

# Login
infisical login
```

## Using Infisical CLI with Docker

```bash
# Inject secrets into a Docker container at runtime
infisical run --env=production -- docker run \
  --name my-app \
  -e NODE_ENV=production \
  my-app:latest

# Get secrets as environment variables
eval $(infisical secrets --env=production --export)

# Inject into docker-compose
infisical run --env=production -- docker-compose up -d
```

## Infisical Agent for Kubernetes

```yaml
# Deploy Infisical agent as sidecar in Kubernetes
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      initContainers:
      - name: infisical-agent
        image: infisical/infisical:latest
        command: ["infisical", "agent"]
        env:
        - name: INFISICAL_TOKEN
          valueFrom:
            secretKeyRef:
              name: infisical-token
              key: token
        volumeMounts:
        - name: secrets-vol
          mountPath: /secrets
      containers:
      - name: app
        image: myapp:latest
        volumeMounts:
        - name: secrets-vol
          mountPath: /run/secrets
      volumes:
      - name: secrets-vol
        emptyDir: {}
```

## Using the Infisical SDK

```python
# Python SDK for dynamic secret injection
from infisical import InfisicalClient

client = InfisicalClient(
    token="your-service-token",
    site_url="https://infisical.example.com"
)

# Get a specific secret
db_password = client.get_secret("DB_PASSWORD", environment="production")

# Get all secrets for an environment
secrets = client.get_all_secrets(environment="production")
for secret in secrets:
    print(f"{secret.secret_name} = {secret.secret_value}")
```

```javascript
// Node.js SDK
const { InfisicalClient } = require('@infisical/sdk');

const client = new InfisicalClient({
    token: process.env.INFISICAL_TOKEN,
    siteUrl: 'https://infisical.example.com'
});

async function getSecrets() {
    const secrets = await client.getAllSecrets({
        environment: 'production',
        projectSlug: 'my-app'
    });
    
    process.env.DB_PASSWORD = secrets.find(s => s.secretName === 'DB_PASSWORD')?.secretValue;
}
```

## Infisical in Portainer Deployment Pipeline

```yaml
# portainer-app-stack.yml - uses Infisical at startup
version: '3.8'

services:
  secrets-init:
    image: infisical/infisical:latest
    command: >
      sh -c "infisical secrets --env=${APP_ENV} --export > /shared-secrets/.env && echo 'Secrets loaded'"
    environment:
      INFISICAL_TOKEN: "${INFISICAL_TOKEN}"
    volumes:
      - shared-secrets:/shared-secrets

  app:
    image: myapp:latest
    depends_on:
      - secrets-init
    env_file:
      - /run/secrets/.env
    volumes:
      - shared-secrets:/run/secrets:ro

volumes:
  shared-secrets:
    driver: local
    driver_opts:
      type: tmpfs  # Use tmpfs for in-memory secrets
      device: tmpfs
```

## Conclusion

Infisical provides a modern, open-source secrets management solution that integrates well with Portainer-managed containers. Its intuitive UI, multi-environment support, and SDK integrations make it accessible to teams of all sizes. Deploying Infisical via Portainer gives you a self-hosted secrets manager with full control over your sensitive data.
