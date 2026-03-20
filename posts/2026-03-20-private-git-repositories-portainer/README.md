# How to Use Private Git Repositories with Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Git, Private Repository, Authentication, GitOps

Description: Learn how to configure Portainer to authenticate with private Git repositories for stack deployments.

## Overview

When your Compose files or Kubernetes manifests live in private Git repositories, Portainer needs credentials to clone and fetch them. Portainer supports username/password (token) and SSH key authentication.

## Method 1: Personal Access Token (Recommended)

Most Git providers support token-based authentication where you use a generated token as the password.

### GitHub

```bash
# Create a Personal Access Token in GitHub:

# Settings > Developer settings > Personal access tokens > Fine-grained tokens
# Grant: Contents (Read-only) for the specific repository

# Use in Portainer:
# Username: your-github-username
# Password: github_pat_XXXXXXXXXXXX
```

### GitLab

```bash
# Create a Deploy Token in GitLab:
# Project > Settings > Repository > Deploy tokens
# Scope: read_repository

# Use in Portainer:
# Username: (the deploy token username, e.g., gitlab+deploy-token-123)
# Password: (the deploy token value)
```

### Bitbucket

```bash
# Create an App Password in Bitbucket:
# Personal settings > App passwords
# Permission: Repositories > Read

# Use in Portainer:
# Username: your-bitbucket-username
# Password: the-app-password
```

## Configuring Credentials in Portainer

When adding a Git-backed stack:

1. Toggle **Authentication** to On.
2. Select **Credentials** type.
3. Enter:
   - **Username**: Your username or token username.
   - **Password/Token**: Your personal access token.
4. Optionally save the credentials for reuse.

## Method 2: SSH Key Authentication

SSH keys work with any Git provider that supports SSH.

### Generating an SSH Key for Portainer

```bash
# Generate an Ed25519 SSH key pair (don't set a passphrase)
ssh-keygen -t ed25519 -f portainer-deploy-key -N "" -C "portainer-deploy"

# Output:
# portainer-deploy      (private key - paste into Portainer)
# portainer-deploy.pub  (public key - add to Git repository)

# View the private key content
cat portainer-deploy
```

### Adding the Deploy Key to GitHub

1. Go to your repository.
2. Navigate to **Settings > Deploy keys**.
3. Click **Add deploy key**.
4. Paste the contents of `portainer-deploy.pub`.
5. Grant read access (no write needed).

### Configuring SSH in Portainer

1. Toggle **Authentication** to On.
2. Select **SSH**.
3. Paste the private key content (`portainer-deploy`).
4. Use the SSH repository URL: `git@github.com:myorg/my-repo.git`

## Repository URL Formats

```bash
# HTTPS (for username/token auth)
https://github.com/myorg/my-repo.git
https://gitlab.com/mygroup/my-project.git

# SSH (for SSH key auth)
git@github.com:myorg/my-repo.git
git@gitlab.com:mygroup/my-project.git

# Self-hosted GitLab
https://gitlab.mycompany.com/team/project.git
git@gitlab.mycompany.com:team/project.git
```

## Saving Credentials for Multiple Stacks

Portainer Business Edition lets you save credentials and reuse them across multiple stacks:

1. Go to **Settings > Credentials**.
2. Add a named credential set.
3. Reference it when creating each Git-backed stack.

## Testing Access Before Deployment

```bash
# Test Git access before configuring Portainer
git clone https://github.com/myorg/private-repo.git --depth=1 /tmp/test-clone
# Use your token as the password when prompted

# Test SSH access
GIT_SSH_COMMAND='ssh -i portainer-deploy-key' \
  git clone git@github.com:myorg/private-repo.git --depth=1 /tmp/test-clone
```

## Conclusion

Private Git repository integration in Portainer is straightforward with either token or SSH authentication. Use deploy tokens or SSH deploy keys (rather than personal credentials) for service account access to follow the principle of least privilege.
