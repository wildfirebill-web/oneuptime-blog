# How to Authenticate with the Portainer API Using Access Tokens

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, API, Access Tokens, Authentication, Automation

Description: Learn how to create and use Portainer access tokens (API keys) for long-lived authentication in automation scripts.

## Access Tokens vs. JWT Tokens

| Feature | JWT Token | Access Token |
|---------|-----------|--------------|
| Lifetime | 8 hours (expires) | Until manually revoked |
| Use case | Interactive sessions | Long-lived automation |
| Creation | Via `/api/auth` | Via Portainer UI |
| Security | Auto-expires | Requires manual rotation |

For CI/CD pipelines and cron jobs, **access tokens are preferred** because they don't expire and don't require storing your password.

## Creating an Access Token in Portainer

1. Log in to Portainer.
2. Click your username in the top right.
3. Select **My account**.
4. Scroll to **Access tokens**.
5. Click **Add access token**.
6. Give it a descriptive name (e.g., `github-actions-deploy`).
7. Copy the token — **you won't see it again**.

## Using an Access Token in API Calls

Access tokens are used identically to JWT tokens — just pass them as the Bearer token:

```bash
# Set your access token
API_TOKEN="ptr_your_access_token_here"
PORTAINER_URL="https://portainer.mycompany.com"

# List all environments using the access token
curl -s "${PORTAINER_URL}/api/endpoints" \
  -H "Authorization: Bearer ${API_TOKEN}" | jq '.'

# Get all stacks
curl -s "${PORTAINER_URL}/api/stacks" \
  -H "Authorization: Bearer ${API_TOKEN}" | jq '.'
```

## Using Access Tokens in Scripts

```bash
#!/bin/bash
# deploy.sh - Redeploy a stack using Portainer API

PORTAINER_URL="https://portainer.mycompany.com"
API_TOKEN="${PORTAINER_API_TOKEN}"  # Set as environment variable
STACK_ID="3"
ENDPOINT_ID="1"

# Update a stack using the access token
curl -s -X PUT \
  "${PORTAINER_URL}/api/stacks/${STACK_ID}?endpointId=${ENDPOINT_ID}" \
  -H "Authorization: Bearer ${API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "pullImage": true,
    "prune": false
  }'
```

## Using Access Tokens in GitHub Actions

```yaml
# .github/workflows/deploy.yml
name: Deploy via Portainer API

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Redeploy stack
        run: |
          curl -s -X POST \
            "${{ secrets.PORTAINER_URL }}/api/webhooks/${{ secrets.PORTAINER_WEBHOOK_TOKEN }}" \
            -H "Authorization: Bearer ${{ secrets.PORTAINER_API_TOKEN }}"
```

Store `PORTAINER_API_TOKEN` as a GitHub Actions secret, not in your code.

## Creating Access Tokens via API

```bash
# Create an access token via the API (using an existing JWT)
curl -X POST "${PORTAINER_URL}/api/users/${USER_ID}/tokens" \
  -H "Authorization: Bearer ${JWT_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"description": "ci-cd-token"}'
```

## Listing and Revoking Access Tokens

In Portainer UI:
1. Go to **My Account > Access tokens**.
2. See all active tokens with their names and creation dates.
3. Click **Remove** next to a token to revoke it immediately.

## Best Practices

- Name tokens descriptively: `github-actions-prod`, `monitoring-cron`, `terraform-provider`.
- Create one token per integration to enable targeted revocation.
- Rotate tokens periodically and after team member changes.
- Never embed tokens in code — use environment variables or secrets managers.

## Conclusion

Portainer access tokens provide stable, long-lived authentication for automation workflows. They're simpler to manage than JWT tokens for CI/CD pipelines because they don't expire and don't require storing a password in your automation environment.
