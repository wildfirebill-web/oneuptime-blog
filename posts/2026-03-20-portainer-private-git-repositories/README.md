# How to Use Private Git Repositories with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, GitOps, Docker, Security, Authentication

Description: Learn how to configure Portainer to authenticate with private Git repositories using personal access tokens and SSH keys for secure stack deployments.

## Introduction

Most production infrastructure repositories are private, requiring authentication to access. Portainer supports private repositories through two methods: username/password (or token-based HTTP authentication) and SSH key-based authentication. This guide covers both approaches for GitHub, GitLab, Gitea, and other Git hosts.

## Prerequisites

- Portainer CE or BE
- A private Git repository containing Docker Compose files
- Repository credentials (PAT token or SSH key)

## Method 1: Personal Access Token (HTTPS)

This is the most common approach and works with all major Git hosts.

### GitHub — Create a Personal Access Token

1. Go to GitHub → **Settings** → **Developer settings** → **Personal access tokens** → **Tokens (classic)**.
2. Click **Generate new token (classic)**.
3. Set a name: `Portainer Deploy Token`
4. Set expiry: 90 days (or no expiry for simplicity)
5. Select scopes:
   - `repo` (full repository access — required for private repos)
6. Click **Generate token**.
7. Copy the token immediately.

```bash
# Test your token
curl -H "Authorization: token ghp_YourTokenHere" \
  https://api.github.com/repos/your-org/your-repo
```

### GitLab — Create a Project Access Token

1. Go to your GitLab project → **Settings** → **Access tokens**.
2. Click **Create project access token**.
3. Name: `Portainer`
4. Role: `Reporter` (minimum needed for read access)
5. Scopes: `read_repository`
6. Click **Create project access token**.

### Configure in Portainer (UI)

1. When creating/editing a Git-connected stack:
2. Enable **Authentication**.
3. Select **Username/Password**.
4. Set:
   - **Username**: your Git username (or `x-token` for tokens-only auth)
   - **Password**: your personal access token

For GitHub specifically:

```
Username: your-github-username
Password: ghp_YourPersonalAccessToken
```

For GitLab:

```
Username: your-gitlab-username  (or oauth2 if using token)
Password: glpat-YourProjectAccessToken
```

### Configure via the Portainer API

```bash
PORTAINER_URL="https://portainer.example.com"
TOKEN="your-portainer-auth-token"
ENDPOINT_ID=1

# Create stack from private Git repository
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "${PORTAINER_URL}/api/stacks/create/standalone/repository?endpointId=${ENDPOINT_ID}" \
  -d '{
    "name": "my-private-app",
    "repositoryURL": "https://github.com/your-org/private-infra",
    "repositoryReferenceName": "refs/heads/main",
    "filePathInRepository": "docker-compose.yml",
    "repositoryAuthentication": true,
    "repositoryUsername": "your-username",
    "repositoryPassword": "ghp_YourPersonalAccessToken",
    "env": []
  }' | jq .
```

## Method 2: SSH Key Authentication

SSH keys are more secure than tokens and are preferred for automated systems.

### Step 1: Generate an SSH Deploy Key

```bash
# Generate a dedicated SSH key for Portainer (no passphrase for automation)
ssh-keygen -t ed25519 -C "portainer-deploy-key" \
  -f ~/.ssh/portainer-deploy-key \
  -N ""  # No passphrase

echo "=== Public Key (add to repository) ==="
cat ~/.ssh/portainer-deploy-key.pub

echo ""
echo "=== Private Key (add to Portainer) ==="
cat ~/.ssh/portainer-deploy-key
```

### Step 2: Add the Deploy Key to Your Repository

**GitHub:**
1. Repository → **Settings** → **Deploy keys** → **Add deploy key**
2. Title: `Portainer Deploy Key`
3. Key: Paste the content of `portainer-deploy-key.pub`
4. Allow write access: Not needed (read-only is sufficient)
5. Click **Add key**

**GitLab:**
1. Repository → **Settings** → **Repository** → **Deploy keys**
2. Title: `Portainer`
3. Key: Paste public key content
4. Grant write permissions: No
5. Click **Add key**

### Step 3: Configure SSH in Portainer (UI)

1. When creating/editing the Git-connected stack:
2. Use the SSH repository URL format:
   - GitHub: `git@github.com:your-org/your-repo.git`
   - GitLab: `git@gitlab.com:your-org/your-repo.git`
   - Gitea: `git@git.company.com:your-org/your-repo.git`
3. Enable **Authentication**.
4. Select **SSH key**.
5. Paste the private key content.
6. Click **Save**.

### Configure via API

```bash
# Read your private key
SSH_PRIVATE_KEY=$(cat ~/.ssh/portainer-deploy-key)

# Create stack with SSH authentication
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "${PORTAINER_URL}/api/stacks/create/standalone/repository?endpointId=${ENDPOINT_ID}" \
  -d "{
    \"name\": \"my-private-app\",
    \"repositoryURL\": \"git@github.com:your-org/private-infra.git\",
    \"repositoryReferenceName\": \"refs/heads/main\",
    \"filePathInRepository\": \"docker-compose.yml\",
    \"repositoryAuthentication\": true,
    \"repositoryGitCredentialID\": 0,
    \"repositoryUsername\": \"\",
    \"repositoryPassword\": $(echo "$SSH_PRIVATE_KEY" | jq -Rs .)
  }" | jq .
```

## Using Portainer's Stored Git Credentials

Portainer BE allows you to store Git credentials centrally and reuse them across multiple stacks:

1. Go to **Settings** → **Credentials** (BE only).
2. Click **Add credentials**.
3. Name: `GitHub Deploy Key`
4. Enter your credentials.
5. When creating stacks, select the stored credential instead of entering it manually.

## Self-Hosted Git (Gitea/GitLab CE) with Self-Signed Certs

For self-hosted Git servers with self-signed TLS:

1. When configuring the repository, enable **Skip TLS verification** (for testing only).
2. For production, use the **CA certificate** option to add your internal CA.

```bash
# Get your internal CA certificate
openssl s_client -connect git.company.com:443 -showcerts </dev/null 2>/dev/null | \
  openssl x509 -outform PEM > internal-ca.pem

# In Portainer stack creation, upload this CA file or paste its content
```

## Security Best Practices

1. Use deploy keys (SSH or PAT) with **read-only** access — write access is not needed
2. Create **dedicated** credentials for Portainer, not your personal account keys
3. Rotate credentials on a schedule (every 6-12 months)
4. Use tokens with **minimal scopes** — only `repo:read` or `read_repository`
5. Name tokens clearly: `portainer-production-deploy-2026-03`
6. Revoke credentials immediately if Portainer is compromised

## Conclusion

Connecting Portainer to private Git repositories is straightforward using either PAT-based HTTPS authentication or SSH deploy keys. SSH keys are preferred for production systems as they can be scoped to specific repositories with read-only access. Store credentials using Portainer's credential manager (BE) or your CI/CD secrets manager, and rotate them regularly to maintain security hygiene.
