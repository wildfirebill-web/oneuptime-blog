# How to Fix Environment Variable Issues on Synology with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Troubleshooting, Synology, Environment Variables, Docker, NAS

Description: Learn how to fix environment variable handling issues when running Portainer on Synology NAS, including DSM version quirks and variable escaping problems.

---

Synology NAS devices run a customized version of DSM (DiskStation Manager) with a modified Docker package. Environment variables in Portainer on Synology can behave unexpectedly due to DSM's string handling and limited shell environment.

## Common Issues on Synology

- Special characters in env vars cause silent failures
- Variables set in Portainer are not visible inside containers
- Multi-line values are silently truncated
- Variables with `$` signs are interpolated unexpectedly

## Step 1: Verify Variables Are Set Correctly

```bash
# SSH into your Synology (enable SSH in DSM Control Panel > Terminal)
# Check if a container received its env vars
docker exec <container-name> env | sort

# If the variable is missing or has wrong value, the issue is in how it was set
```

## Step 2: Avoid Special Characters in Variable Values

Portainer on Synology passes env vars through DSM's Docker manager. Characters that need escaping:

```bash
# Problematic: passwords with special characters
DB_PASS=p@$$w0rd!          # $ signs cause issues

# Safer: use only alphanumeric and limited special chars
DB_PASS=pAtSword123        # Works reliably

# If you must use special characters, use an .env file or Docker secrets
```

## Step 3: Use Stack Files Instead of Container UI

For complex environment configurations, use a stack (Compose file) instead of the container creation UI:

```yaml
# Place this in Portainer's stack editor
version: "3.8"
services:
  myapp:
    image: myimage:latest
    environment:
      # Portainer's stack editor handles env vars more reliably than the container UI
      DB_HOST: "postgres"
      DB_PORT: "5432"
      DB_PASS: "your-password"
```

## Step 4: Use .env Files for Sensitive Values

For sensitive values, place an `.env` file in the Portainer stack storage directory:

```bash
# Create an env file on the Synology volume
cat > /volume1/docker/myapp.env << 'EOF'
DB_PASS=supersecret
API_KEY=abc123
EOF

# Reference it in the stack
# Note: on Synology this must be an absolute path
```

## Step 5: Fix DSM Docker Package Version

Older DSM Docker packages have bugs with env var handling. Update via DSM:

1. Open **Package Center**.
2. Find **Docker**.
3. Click **Update** if available.

## Step 6: Check for Variable Shadowing

DSM containers inherit some environment variables from the DSM host. Verify no shadowing:

```bash
docker exec <container-name> env | grep -i your_variable_name
# If you see unexpected values, DSM may be injecting conflicting variables
# Prefix your variable names to avoid conflicts: APP_DB_HOST instead of DB_HOST
```
