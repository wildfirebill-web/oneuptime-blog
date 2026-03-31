# How to Manage Git Credentials in Portainer Business Edition

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Git, Business Edition, GitOps, Credential, Security

Description: Store and manage Git credentials in Portainer Business Edition for secure GitOps deployments from private repositories.

## Introduction

Portainer Business Edition allows you to store Git credentials centrally and reuse them across multiple stacks and environments. Instead of entering credentials each time you deploy from a private repository, you save them once and reference them by name. This guide covers creating, using, and managing Git credentials in Portainer BE.

## Prerequisites

- Portainer Business Edition (CE does not support saved Git credentials)
- Access to a private Git repository (GitHub, GitLab, Bitbucket, Azure DevOps, etc.)
- Admin or appropriate user permissions

## Types of Git Authentication

Portainer supports:
1. **Username + Personal Access Token (PAT)** - Most common, works with all major Git providers
2. **Username + Password** - Deprecated by most providers, not recommended
3. **SSH Key** - For SSH-based Git URLs

## Step 1: Generate a Personal Access Token

### GitHub PAT

1. Go to **GitHub** → **Settings** → **Developer settings** → **Personal access tokens** → **Fine-grained tokens**
2. Click **Generate new token**
3. Set expiration and permissions:
   - Repository access: Select specific repositories or all
   - Permissions: `Contents: Read` at minimum
4. Copy the token - it's shown once

### GitLab PAT

1. Go to **GitLab** → **User Settings** → **Access Tokens**
2. Create a token with `read_repository` scope
3. Copy the token

## Step 2: Add Git Credentials in Portainer

1. Log in to Portainer Business Edition
2. Go to **Account** → **Git credentials** (or **Settings** → **Git credentials** for shared credentials)
3. Click **Add credentials**
4. Fill in:

```text
Name:     my-github-credentials
Username: your-github-username
Personal Access Token: ghp_your_token_here
```

5. Click **Save credentials**

## Step 3: Use Saved Credentials in a Stack

When creating or editing a stack from a Git repository:

1. Select **Repository** as the source
2. Enter the repository URL (e.g., `https://github.com/yourorg/your-private-repo`)
3. Under **Authentication**, select **Use saved credentials**
4. Choose your saved credential set from the dropdown
5. Configure branch and compose file path
6. Deploy

## Managing Credentials via the API

```bash
# Get admin token

TOKEN=$(curl -s -X POST \
  https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"adminpassword"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# List Git credentials
curl -s \
  -H "Authorization: Bearer $TOKEN" \
  https://portainer.example.com/api/gitcredentials \
  | python3 -m json.tool

# Create Git credentials
curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/gitcredentials \
  -d '{
    "name": "github-ci-account",
    "username": "ci-bot-user",
    "password": "ghp_your_personal_access_token"
  }'

# Update credentials (e.g., rotate PAT)
CRED_ID=1
curl -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "https://portainer.example.com/api/gitcredentials/${CRED_ID}" \
  -d '{
    "name": "github-ci-account",
    "username": "ci-bot-user",
    "password": "ghp_new_rotated_token"
  }'

# Delete credentials
curl -X DELETE \
  -H "Authorization: Bearer $TOKEN" \
  "https://portainer.example.com/api/gitcredentials/${CRED_ID}"
```

## Rotating Credentials

When a PAT expires or is rotated:

1. Generate a new PAT in your Git provider
2. Update the credential in Portainer via UI or API
3. All stacks using the saved credential automatically use the new token
4. Revoke the old PAT in your Git provider

This is the key advantage of saved credentials - updating in one place updates all deployments.

## Sharing Credentials vs. Per-User Credentials

| Type | Scope | Created By |
|------|-------|-----------|
| User credentials | Only visible to the creating user | Any user |
| Shared credentials | Visible to admin and team members | Admin |

For team environments, admins can create shared credentials that all team members can select when deploying.

## Security Best Practices

- Use fine-grained PATs with the minimum required permissions (read-only for deployments)
- Set expiration dates on PATs and rotate them regularly
- Use a dedicated service account in your Git provider (not a personal account)
- Never use credentials that have write access to production branches in automated deployments

## Conclusion

Portainer Business Edition's Git credential management simplifies GitOps workflows by centralizing authentication. Instead of managing credentials per-stack, you maintain them in one place and rotate them without touching individual stack configurations. This makes credential hygiene practical even when you have dozens of Git-backed stacks.
