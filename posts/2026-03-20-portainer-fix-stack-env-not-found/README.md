# How to Fix stack.env Not Found Errors in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Stacks, Troubleshooting, DevOps

Description: Learn how to diagnose and fix "stack.env not found" and related environment file errors when deploying or updating stacks in Portainer.

## Introduction

The `stack.env not found` error appears when Portainer or Docker Compose cannot locate the environment file referenced in a stack configuration. This commonly happens when a Compose file references an `env_file` path that doesn't exist in the deployment context, when the Portainer stack's associated `.env` file is missing, or after a Portainer data migration. Understanding where Portainer stores stack files helps diagnose and resolve these errors quickly.

## Prerequisites

- Portainer with a stack experiencing the env not found error
- Shell access to the Portainer host

## Common Error Messages

```
# Error 1: env_file reference in Compose:
Error response from daemon: open /data/compose/12/stack.env: no such file or directory

# Error 2: Portainer's internal stack storage:
failed to get stack: stack not found in: /data/compose/12/

# Error 3: Git-based stack missing env file:
failed to deploy stack, git repository does not contain stack.env

# Error 4: Variable undefined:
variable "DB_PASSWORD" is not set. Defaulting to a blank string.
```

## Step 1: Understand Portainer's Stack File Storage

Portainer stores stack files in its data volume:

```bash
# Find where Portainer stores stack files:
docker exec portainer ls /data/compose/

# Each stack gets a numbered directory:
# /data/compose/1/  → stack ID 1
# /data/compose/12/ → stack ID 12

# Inside each directory:
docker exec portainer ls /data/compose/12/
# docker-compose.yml  (the Compose file)
# stack.env           (environment variables — if configured)
```

## Step 2: Identify the Missing File

```bash
# Check if the stack.env file exists:
docker exec portainer ls -la /data/compose/12/

# If stack.env is missing but the stack has environment variables configured:
# → Portainer's data was corrupted or the file was deleted

# Check the Compose file for env_file references:
docker exec portainer cat /data/compose/12/docker-compose.yml | grep env_file
```

## Step 3: Fix Missing stack.env (Portainer Data Issue)

If the `stack.env` file was deleted from Portainer's data volume:

```bash
# 1. Find the stack ID in Portainer:
#    Navigate to Stacks → click the stack → note the URL: /stacks/12

# 2. Check what's in the compose directory:
docker exec portainer ls /data/compose/12/

# 3. Recreate the stack.env file with the correct variables:
docker exec portainer sh -c 'cat > /data/compose/12/stack.env << EOF
DB_PASSWORD=your_password
IMAGE_TAG=v1.2.3
LOG_LEVEL=info
EOF'

# 4. Update the stack in Portainer to re-read the environment:
#    Portainer UI: Stacks → stack name → Update the stack
```

## Step 4: Fix env_file Reference in Compose YAML

If your Compose file has an `env_file` directive:

```yaml
# Problem: references a file that may not exist in Portainer's context
services:
  api:
    image: myorg/api:latest
    env_file:
      - ./stack.env    # This must exist in the same directory as docker-compose.yml
```

Solutions:

**Option A: Remove the `env_file` directive and use the Portainer Variables UI**:

```yaml
# Corrected: use environment directly (variables set via Portainer UI)
services:
  api:
    image: myorg/api:latest
    environment:
      - DB_PASSWORD=${DB_PASSWORD}
      - IMAGE_TAG=${IMAGE_TAG}
      - LOG_LEVEL=${LOG_LEVEL}
```

**Option B: For Git-based stacks, include the env file in the repository**:

```bash
# Repository structure:
my-infra/
├── docker-compose.yml    # References ./stack.env
└── stack.env             # Non-secret defaults committed to Git
```

Note: Only commit non-sensitive defaults to Git. Override sensitive values in Portainer's environment variables section.

## Step 5: Fix for Git-Based Stacks

If the Git repository is missing the referenced env file:

```bash
# Check if stack.env exists in the Git repository:
git ls-files | grep env

# Add a stack.env with default values (no secrets):
cat > stack.env << 'EOF'
# Stack environment defaults — override sensitive values in Portainer
LOG_LEVEL=info
APP_PORT=8080
WORKERS=2
DB_PORT=5432
EOF

git add stack.env
git commit -m "Add stack.env defaults for Portainer deployment"
git push
```

Then in Portainer, set sensitive variables (passwords, tokens) via the UI — they override the file values.

## Step 6: Recreate the Stack to Fix Persistent Errors

If the stack is in a broken state that won't update:

```bash
# 1. Copy the current Compose YAML from Portainer UI editor
# 2. Note all environment variables

# 3. Remove the broken stack (preserving volumes):
#    Portainer UI: Stacks → check the stack → Remove (no volumes option)

# 4. Redeploy as a new stack:
#    Portainer UI: Stacks → Add stack → paste Compose YAML
#    Add environment variables
#    Deploy

# The containers will reconnect to existing volumes by name
```

## Step 7: Verify Environment Variables After Fix

```bash
# Confirm all expected variables are present in the container:
docker exec myapp_api_1 env | sort

# Check for any blank/undefined variables:
docker exec myapp_api_1 env | grep -E "^[A-Z_]+=\s*$"
# Should return nothing (no blank variables)

# Test the application responds correctly:
curl http://localhost:8080/health
```

## Conclusion

The `stack.env not found` error has three distinct causes: a missing `env_file` reference in the Compose YAML, a corrupted or deleted Portainer data volume file, or a Git repository missing the referenced file. The cleanest fix is to remove the `env_file` directive from the Compose YAML entirely and manage all environment variables through Portainer's UI. For Git-based stacks, include a non-sensitive `stack.env` with defaults in the repository and override sensitive values in Portainer. If the Portainer data volume is the issue, recreate the `stack.env` file directly in the container or redeploy the stack from scratch.
