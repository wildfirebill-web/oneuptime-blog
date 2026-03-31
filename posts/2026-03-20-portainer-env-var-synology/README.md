# How to Fix Environment Variable Issues on Synology with Portainer - Env Var

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Synology, NAS, Troubleshooting, Environment Variable

Description: Resolve environment variable handling problems when running Portainer on Synology NAS, including DSM Docker differences, env file parsing issues, and variable escaping.

## Introduction

Synology NAS devices run Docker through DSM (DiskStation Manager), which uses an older Docker version and has some non-standard behaviors. When running Portainer on Synology, environment variable issues are common - values containing special characters get corrupted, `.env` files don't parse correctly, or variables passed to containers via Portainer's UI behave differently than expected.

## Understanding the Synology Docker Environment

Synology DSM Docker:
- Runs Docker in a compatibility layer
- May use an older Docker Engine version
- Has file permission restrictions in some directories
- DSM's built-in Docker UI and Portainer can conflict

## Step 1: Verify Portainer Is Running Correctly on Synology

```bash
# SSH into your Synology NAS

ssh admin@synology-ip

# Check Docker version
docker version

# Check Portainer status
docker ps | grep portainer

# Check Portainer logs
docker logs portainer --tail 50
```

## Step 2: Fix Special Characters in Environment Variables

On Synology, environment variables with special characters often get mangled:

```bash
# PROBLEMATIC: Variables with special chars in Portainer UI
# PASSWORD=my$ecret  ← $ gets interpreted as variable
# SECRET=abc"def"ghi ← quotes get stripped

# FIX: Use .env files instead of inline values in the Portainer UI
```

Create an `.env` file on the Synology:

```bash
# Create a secure directory for environment files
mkdir -p /volume1/docker/myapp/
touch /volume1/docker/myapp/.env
chmod 600 /volume1/docker/myapp/.env

# Write the env file
cat > /volume1/docker/myapp/.env << 'EOF'
DB_PASSWORD=my$ecret!with@special#chars
SECRET_KEY=abc"def"ghi
API_KEY=Bearer eyJhbGciOiJSUzI1NiIsImtp
EOF
```

## Step 3: Use .env Files in Portainer Stacks

When creating a stack in Portainer on Synology:

```yaml
version: "3.8"
services:
  myapp:
    image: myapp:latest
    env_file:
      # Use absolute path on Synology
      - /volume1/docker/myapp/.env
    # Do NOT inline sensitive values here
```

Or reference specific variables:

```yaml
services:
  myapp:
    image: myapp:latest
    environment:
      # Reference from .env file loaded via env_file
      DB_PASSWORD: "${DB_PASSWORD}"
      SECRET_KEY: "${SECRET_KEY}"
```

## Step 4: Fix Variable Escaping in Portainer Stack Editor

In Portainer's web editor, dollar signs in values need escaping:

```yaml
# In Portainer's web editor:
services:
  myapp:
    environment:
      # Use $$ for literal $ in Portainer web editor
      MY_VAR: "value_with_$$dollar"

      # Or use the .env file approach (preferred)
```

## Step 5: Fix Portainer Stack Variables on Synology

When using Portainer's "Environment variables" feature for stacks:

```bash
# Portainer supports substitution variables at the stack level
# Go to: Stacks → Create Stack → scroll down to "Environment variables"
# Add variables there instead of hardcoding in the compose file

# In compose file, reference like:
environment:
  DB_PASSWORD: ${STACK_DB_PASSWORD}
```

## Step 6: Fix Synology Volume Path Issues

On Synology, Docker volume paths differ from standard Linux:

```bash
# Standard Linux path
-v /home/user/data:/data

# Synology path (volumes are on /volume1, /volume2, etc.)
-v /volume1/docker/myapp/data:/data

# Verify the path exists
ls -la /volume1/docker/myapp/data/

# Create if missing
mkdir -p /volume1/docker/myapp/data
```

## Step 7: Fix newline Characters in Variables

Synology's Docker may handle newlines in env values differently:

```bash
# Check for hidden characters
cat -A /volume1/docker/myapp/.env | grep -v "^$"
# ^ symbols at end of lines indicate Windows-style CRLF line endings

# Fix line endings
sed -i 's/\r//' /volume1/docker/myapp/.env

# Or use tr
tr -d '\r' < /volume1/docker/myapp/.env > /volume1/docker/myapp/.env.fixed
mv /volume1/docker/myapp/.env.fixed /volume1/docker/myapp/.env
```

## Step 8: Verify Environment Variables Inside Container

```bash
# Start a shell in the running container
docker exec -it myapp-container sh

# List all environment variables
env | sort

# Check for specific variable
echo $DB_PASSWORD

# Verify no truncation
echo "${SECRET_KEY}" | wc -c  # Count characters

exit
```

## Step 9: Fix DSM Docker Compatibility Issues

On Synology DSM, there can be version conflicts between DSM's built-in Docker and Portainer:

```bash
# Check DSM Docker version
docker version

# If DSM uses Docker 20.x or older with Portainer 2.20+
# There may be API compatibility issues

# Solution: Use a specific Portainer version compatible with your DSM Docker
# Check Portainer release notes for compatibility

# Example: Use an older Portainer version
docker run -d \
  -p 9000:9000 \
  -p 9443:9443 \
  --name portainer \
  --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:2.19.5  # Specific version for older Docker
```

## Step 10: Use Docker Compose Validation Before Deploying

```bash
# Validate your compose file locally before uploading to Portainer
docker compose config -f /volume1/docker/myapp/docker-compose.yml

# Test environment variable substitution
docker compose config --env-file /volume1/docker/myapp/.env
```

## Conclusion

Environment variable issues on Synology with Portainer are primarily caused by special character handling in the Portainer web editor and CRLF line endings in `.env` files. The most reliable approach is to store sensitive environment variables in a `.env` file on a Synology volume and reference it via `env_file` in your compose stack - avoiding the need to type special characters in the Portainer UI altogether.
