# How to Use the --admin-password Flag in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, CLI, Configuration, Security, Self-Hosted

Description: Use the --admin-password and --admin-password-file flags to pre-configure Portainer's admin account at startup, avoiding the 5-minute initialization window.

## Introduction

The `--admin-password` flag allows you to set the Portainer admin password at container startup rather than waiting for the initialization UI. This is essential for automated deployments, CI/CD pipelines, and any scenario where you need Portainer to be ready without manual browser interaction.

## How --admin-password Works

When you pass `--admin-password` with a bcrypt-hashed password:
1. Portainer creates the admin account automatically on startup
2. The 5-minute initialization window is bypassed
3. You can log in immediately after the container starts

## Step 1: Generate a Bcrypt Password Hash

```bash
# Method 1: Using Docker with httpd (most portable)

docker run --rm httpd:2.4-alpine htpasswd -nbB admin yourpassword | cut -d ':' -f 2
# Output: $2y$05$...

# Method 2: Using htpasswd if installed
htpasswd -nbB admin yourpassword | cut -d ':' -f 2

# Method 3: Using Python (if available)
python3 -c "import bcrypt; print(bcrypt.hashpw(b'yourpassword', bcrypt.gensalt(rounds=5)).decode())"

# Method 4: Using openssl (only for specific bcrypt implementations)
# Note: openssl doesn't natively support bcrypt - use the above methods

# Save the hash for use
HASH=$(docker run --rm httpd:2.4-alpine htpasswd -nbB admin yourpassword | cut -d ':' -f 2)
echo "Hash: $HASH"
```

## Step 2: Use --admin-password on the Command Line

```bash
# Start Portainer with a pre-set admin password
docker run -d \
  -p 9000:9000 \
  -p 9443:9443 \
  --name portainer \
  --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --admin-password='$2y$05$8qOfvkl7D4FtcC/eCIbVGeFNQtYjC6.gg5bflnEsOxOinqPgXHzaC'

# Note: Single quotes prevent shell from interpreting $ characters
```

## Step 3: Use --admin-password with a Variable

```bash
# Generate the hash first
HASHED_PASSWORD=$(docker run --rm httpd:2.4-alpine htpasswd -nbB admin yourpassword | cut -d ':' -f 2)

echo "Hash generated: $HASHED_PASSWORD"

# Start Portainer with the hashed password
docker run -d \
  -p 9000:9000 \
  -p 9443:9443 \
  --name portainer \
  --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --admin-password="$HASHED_PASSWORD"
```

## Step 4: Use --admin-password-file (More Secure)

The `--admin-password-file` flag reads the hash from a file, keeping it out of the process list:

```bash
# Create the password hash file
docker run --rm httpd:2.4-alpine htpasswd -nbB admin yourpassword | \
  cut -d ':' -f 2 > /tmp/portainer-passwd

chmod 600 /tmp/portainer-passwd

# Start Portainer with password file
docker run -d \
  -p 9000:9000 \
  -p 9443:9443 \
  --name portainer \
  --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  -v /tmp/portainer-passwd:/tmp/portainer-passwd:ro \
  portainer/portainer-ce:latest \
  --admin-password-file=/tmp/portainer-passwd
```

## Step 5: Use in Docker Compose

```yaml
version: "3.8"
services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: unless-stopped
    # Use double $$ to escape $ in compose files
    command: --admin-password='$$2y$$05$$8qOfvkl7D4FtcC/eCIbVGeFNQtYjC6.gg5bflnEsOxOinqPgXHzaC'
    ports:
      - "9000:9000"
      - "9443:9443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data

volumes:
  portainer_data:
```

Or use the file approach with compose:

```yaml
version: "3.8"
services:
  portainer:
    image: portainer/portainer-ce:latest
    command: --admin-password-file=/run/secrets/portainer-admin-password
    ports:
      - "9000:9000"
      - "9443:9443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    secrets:
      - portainer-admin-password

secrets:
  portainer-admin-password:
    file: ./portainer-password-hash.txt

volumes:
  portainer_data:
```

## Step 6: Use Docker Secrets (Docker Swarm)

```bash
# Create the secret in Swarm
docker run --rm httpd:2.4-alpine htpasswd -nbB admin yourpassword | \
  cut -d ':' -f 2 | \
  docker secret create portainer-admin-password -

# Deploy Portainer with the secret
docker service create \
  --name portainer \
  --publish 9000:9000 \
  --publish 9443:9443 \
  --mount type=volume,src=portainer_data,dst=/data \
  --mount type=bind,src=/var/run/docker.sock,dst=/var/run/docker.sock \
  --secret portainer-admin-password \
  --constraint 'node.role == manager' \
  portainer/portainer-ce:latest \
  --admin-password-file=/run/secrets/portainer-admin-password
```

## Step 7: Automate Portainer Deployment

```bash
#!/bin/bash
# deploy-portainer.sh
# Full automated Portainer deployment

set -e

PORTAINER_PASSWORD="${1:-changeme}"
PORTAINER_VERSION="${2:-latest}"

echo "Generating password hash..."
HASH=$(docker run --rm httpd:2.4-alpine \
  htpasswd -nbB admin "$PORTAINER_PASSWORD" | cut -d ':' -f 2)

echo "Stopping existing Portainer if running..."
docker stop portainer 2>/dev/null || true
docker rm portainer 2>/dev/null || true

echo "Starting Portainer $PORTAINER_VERSION..."
docker run -d \
  -p 9000:9000 \
  -p 9443:9443 \
  --name portainer \
  --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  "portainer/portainer-ce:$PORTAINER_VERSION" \
  --admin-password="$HASH"

echo "Waiting for Portainer to start..."
sleep 5

# Test login
STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
  -X POST http://localhost:9000/api/auth \
  -H "Content-Type: application/json" \
  -d "{\"Username\":\"admin\",\"Password\":\"$PORTAINER_PASSWORD\"}")

if [ "$STATUS" = "200" ]; then
  echo "Portainer deployed successfully!"
  echo "URL: http://$(hostname -I | awk '{print $1}'):9000"
  echo "Password: $PORTAINER_PASSWORD"
else
  echo "Warning: Login test returned HTTP $STATUS"
fi
```

## Important Notes

```bash
# The --admin-password flag ONLY takes effect on first startup
# (when portainer.db doesn't exist or has no admin user)
#
# On subsequent restarts, the stored password in portainer.db takes precedence
#
# To RESET the admin password, you must:
# 1. Stop Portainer
# 2. Delete portainer.db (or the entire volume)
# 3. Restart with --admin-password
#
# OR use the Portainer API to change the password:
TOKEN=$(curl -s -X POST http://localhost:9000/api/auth \
  -H "Content-Type: application/json" \
  -d '{"Username":"admin","Password":"oldpassword"}' | jq -r .jwt)

curl -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  http://localhost:9000/api/users/1/passwd \
  -d '{"Password":"newpassword"}'
```

## Conclusion

The `--admin-password` flag is the recommended way to set the Portainer admin password in automated deployments. Use `--admin-password-file` for better security as it keeps the hash out of the process list. Remember that this flag only configures the password on initial database creation - for existing installations, use the Portainer UI or API to change passwords.
