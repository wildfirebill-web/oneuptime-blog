# How to Deploy Stacks from a Git Repository in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Git, Stack, Deployment, GitOps

Description: Learn how to deploy and manage Docker Compose stacks directly from a Git repository in Portainer.

## Overview

Portainer can fetch your Docker Compose file directly from a Git repository and deploy it as a stack. This keeps your stack definitions version-controlled in Git rather than pasted into the Portainer UI.

## Deploying from a Public Repository

1. Go to **Stacks > Add stack**.
2. Enter a stack name.
3. Select **Git repository** as the build method.
4. Enter:
   - **Repository URL**: `https://github.com/myorg/my-app`
   - **Repository reference**: `refs/heads/main` (or `refs/tags/v1.0.0`)
   - **Compose file path**: `docker-compose.yml`
5. Click **Deploy the stack**.

## Deploying from a Private Repository

For private repositories, enable authentication:

1. Toggle **Authentication** to On.
2. Choose the credential type:
   - **Username/password**: Use a personal access token as the password.
   - **SSH key**: Paste your SSH private key.

```bash
# For GitHub, use a Personal Access Token (PAT) as the password

# Username: your-github-username
# Password: ghp_yourpersonalaccesstoken

# For GitLab, use a Deploy Token
# Username: deploy-token-username
# Password: <deploy-token-value>
```

## Reference Formats

Portainer accepts various Git reference formats:

```bash
refs/heads/main              # Branch
refs/heads/develop           # Another branch
refs/tags/v2.0.0             # Tag
abc123def456                 # Specific commit SHA
```

## Compose File Path Options

```bash
docker-compose.yml           # Root of repository
deploy/docker-compose.yml    # In a subdirectory
production/stack.yml         # Custom filename
```

## Handling Multiple Compose Files

For multi-environment setups, maintain separate compose files in the same repository:

```text
my-repo/
├── docker-compose.yml          # Base configuration
├── docker-compose.prod.yml     # Production overrides
└── docker-compose.staging.yml  # Staging overrides
```

Deploy the production stack using `docker-compose.prod.yml` as the Compose file path.

## Viewing the Current Git State

After deployment, Portainer shows:
- Which Git ref (branch/tag/commit) is deployed.
- When the last update occurred.
- A link to view changes.

## Redeploying After a Git Push

When you push changes to the configured branch:

- **With polling enabled**: Portainer auto-detects within the polling interval (default: 5 minutes).
- **With webhooks enabled**: Git triggers Portainer immediately.
- **Manual**: Click **Pull and redeploy** in the Portainer UI.

```bash
# Manually trigger a stack redeploy via Portainer API
curl -X POST "${PORTAINER_URL}/api/stacks/${STACK_ID}/git/redeploy?endpointId=${ENDPOINT_ID}" \
  -H "Authorization: Bearer ${API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"pullImage": true}'
```

## Conclusion

Deploying stacks from Git repositories is the foundation of GitOps with Portainer. It ensures your deployed configuration is always traceable to a Git commit, enabling audits, rollbacks, and collaborative workflow reviews via pull requests.
