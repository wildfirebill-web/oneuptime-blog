# How to Create a Stack from a Git Repository in Portainer - From

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Stack, Git, DevOps

Description: Learn how to deploy Docker Compose stacks directly from a Git repository in Portainer, enabling GitOps workflows with version-controlled configurations.

## Introduction

Deploying stacks from a Git repository is the recommended production approach for Portainer stack management. Your Docker Compose files live in version control, changes are tracked with commit history, and Portainer can automatically poll for updates and redeploy when the repository changes. This enables GitOps workflows where the Git repository is the single source of truth for your infrastructure.

## Prerequisites

- Portainer installed with a connected Docker environment
- A Git repository containing a `docker-compose.yml` file (GitHub, GitLab, Bitbucket, or self-hosted)
- Repository access credentials (for private repos)

## Step 1: Prepare Your Git Repository

Structure your repository to include the Compose file and any supporting configs:

```text
my-app-infra/
├── docker-compose.yml        # Main stack definition
├── docker-compose.prod.yml   # Production overrides (optional)
├── .env.example              # Example environment variables
├── nginx/
│   └── nginx.conf
└── README.md
```

Example `docker-compose.yml` in the repository:

```yaml
version: "3.8"

services:
  web:
    image: myorg/web:${IMAGE_TAG:-latest}
    restart: unless-stopped
    ports:
      - "80:80"
    networks:
      - app-net
    environment:
      - API_URL=${API_URL}

  api:
    image: myorg/api:${IMAGE_TAG:-latest}
    restart: unless-stopped
    networks:
      - app-net
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_URL=${REDIS_URL}

  postgres:
    image: postgres:15-alpine
    restart: unless-stopped
    networks:
      - app-net
    environment:
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data

networks:
  app-net:
    driver: bridge

volumes:
  postgres_data:
```

## Step 2: Generate a Deploy Key (Private Repos)

For private repositories, create a deploy key:

```bash
# Generate an SSH key pair:

ssh-keygen -t ed25519 -C "portainer-deploy" -f portainer_deploy_key -N ""

# Add the PUBLIC key to your repository:
# GitHub: Settings → Deploy keys → Add deploy key
cat portainer_deploy_key.pub

# Keep the PRIVATE key to paste into Portainer:
cat portainer_deploy_key
```

## Step 3: Create the Stack from Git

1. Navigate to **Stacks** → **Add stack**.
2. Enter the stack **Name**: `my-app`.
3. Select **Repository** as the build method.
4. Configure the Git settings:

```text
Repository URL:  https://github.com/myorg/my-app-infra
                 (or git@github.com:myorg/my-app-infra for SSH)
Repository ref:  refs/heads/main
Compose path:    docker-compose.yml
```

5. For private repositories, enable **Authentication**:
   - For HTTPS: enter your username and a personal access token.
   - For SSH: paste the private key content.

6. Click **Deploy the stack**.

## Step 4: Set Environment Variables

In the **Environment variables** section below the Git configuration:

```text
IMAGE_TAG        v1.2.3
API_URL          https://api.example.com
DATABASE_URL     postgresql://user:pass@postgres:5432/mydb
DB_PASSWORD      securepassword
REDIS_URL        redis://redis:6379
```

These are injected at deploy time and override any `.env` file in the repo.

## Step 5: Use a Specific Branch or Tag

Target specific branches or tags for environment-specific deployments:

```text
# For production (from main branch):
Repository ref: refs/heads/main

# For staging (from develop branch):
Repository ref: refs/heads/develop

# For a pinned release:
Repository ref: refs/tags/v1.2.3

# For a specific commit:
Repository ref: <40-char commit SHA>
```

## Step 6: Enable Auto-Update (Polling)

For automatic redeployment when the repository changes:

1. In the stack creation form, enable **Automatic updates**.
2. Set the polling interval: `5m` (every 5 minutes).
3. Enable **Force re-pull images** if you want to always pull the latest image tags.

Portainer will check the repository at the interval and redeploy if the commit hash has changed.

## Step 7: Verify and Manage

After deployment:

```bash
# Check containers are running:
docker ps --filter "label=com.docker.compose.project=my-app"

# View which Git commit is deployed:
# In Portainer: Stacks → my-app → shows current commit SHA

# Trigger a manual update (after pushing to repo):
# Portainer UI: Stacks → my-app → Pull and redeploy
```

## Conclusion

Deploying stacks from Git repositories is the production-grade approach to stack management in Portainer. Your Compose files gain version history, peer review via pull requests, and rollback capability. Combined with Portainer's auto-update polling, your infrastructure automatically stays synchronized with your Git repository - achieving a GitOps workflow without requiring a dedicated CD tool. Use environment variables in Portainer to inject secrets that should not be committed to the repository.
