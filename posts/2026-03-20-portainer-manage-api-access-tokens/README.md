# How to Manage Your API Access Tokens in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, API, Access Tokens, Automation, Security, Docker, REST API

Description: Learn how to create and manage Portainer API access tokens for automating deployments and integrating with CI/CD pipelines without using username/password authentication.

---

API access tokens in Portainer provide a secure way to authenticate API calls without exposing your password. They are ideal for CI/CD pipelines, automation scripts, and integrations that need programmatic access to Portainer.

## Creating an API Access Token

1. Log in to Portainer.
2. Click your username → **My Account**.
3. Scroll to **Access tokens**.
4. Click **Add access token**.
5. Enter a descriptive name (e.g., "GitHub Actions Deployment").
6. Click **Add access token**.
7. **Copy the token immediately** — it is shown only once.

## Using the Token in API Calls

```bash
# Authenticate with a token instead of username/password
TOKEN="ptr_xxxxxxxxxxxxxxxxxxxxxxxxx"

# List all environments
curl -H "X-API-Key: $TOKEN" \
  http://localhost:9000/api/endpoints

# Deploy or update a stack
curl -X PUT \
  -H "X-API-Key: $TOKEN" \
  -H "Content-Type: application/json" \
  http://localhost:9000/api/stacks/1 \
  -d '{"StackFileContent": "..."}'
```

## Using in GitHub Actions

Store the token as a GitHub Secret and use it in workflows:

```yaml
# .github/workflows/deploy.yml
name: Deploy to Portainer

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Redeploy stack via Portainer webhook
        run: |
          curl -X POST \
            -H "X-API-Key: ${{ secrets.PORTAINER_TOKEN }}" \
            "${{ secrets.PORTAINER_URL }}/api/stacks/1/git/redeploy" \
            -H "Content-Type: application/json" \
            -d '{"env":[],"prune":false}'
```

## Managing Multiple Tokens

Create separate tokens for each integration:

| Token Name | Used By | Scope |
|---|---|---|
| GitHub Actions | CI/CD pipeline | Stack redeployment |
| Monitoring Script | Cron job | Read-only status checks |
| Terraform Provider | IaC | Full management |

Separate tokens let you revoke access for one integration without affecting others.

## Revoking a Token

1. Go to **My Account > Access tokens**.
2. Find the token by name.
3. Click the trash icon to revoke it.

Revoked tokens immediately stop working in all API calls.

## Token Security Best Practices

```bash
# Store tokens in environment variables, never in code
export PORTAINER_TOKEN="ptr_xxx"

# Use GitHub/GitLab secrets for CI/CD (not plaintext in YAML)
# Use a secrets manager (Vault, AWS SSM) for production automation

# Set the minimum required access for the token's use case
# Read-only monitoring does not need write permissions
```
