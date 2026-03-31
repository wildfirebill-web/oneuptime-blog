# How to Store Git Credentials in Portainer User Settings

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Git, Credential, User Setting, GitOps, Security, Stack

Description: Learn how to save and manage Git credentials in Portainer user settings for automatically authenticating to private Git repositories when deploying stacks.

---

Portainer can deploy stacks directly from Git repositories. For private repositories, you need to provide credentials. Storing them in user settings lets you reuse them without re-entering on every deployment.

## Saving Git Credentials

1. Log in to Portainer.
2. Click your username → **My Account**.
3. Scroll to **Git credentials**.
4. Click **Add credentials**.
5. Enter a **Name** (e.g., "GitHub Personal"), **Username**, and **Personal Access Token** (recommended over password).
6. Click **Save credentials**.

## Using a Personal Access Token (Recommended)

GitHub, GitLab, and Bitbucket support PATs, which can be scoped to only repository read access:

**GitHub PAT:**
1. Go to `github.com` → **Settings → Developer settings → Personal access tokens → Tokens (classic)**.
2. Generate a new token with `repo` scope (or `read:repo` for read-only).
3. Copy the token and use it as the password in Portainer.

**GitLab PAT:**
1. Go to `gitlab.com` → **User Settings → Access Tokens**.
2. Create a token with `read_repository` scope.

## Deploying a Stack from a Private Repository

After saving credentials:

1. In Portainer go to **Stacks > Add Stack**.
2. Choose **Repository** as the build method.
3. Enter the repository URL (e.g., `https://github.com/myorg/private-repo`).
4. Under **Authentication**, select **Credentials** and choose your saved credentials.
5. Set the branch and compose file path.
6. Click **Deploy the stack**.

## SSH Key Authentication

For Git over SSH, add an SSH private key instead of a token:

1. In **My Account > Git credentials**, click **Add credentials**.
2. Select **SSH** as the type.
3. Paste your private key.

On the repository side, add the corresponding public key as a deploy key.

## Credential Security

Portainer stores Git credentials encrypted in its BoltDB database. The encryption key is derived from the installation-specific `SECRET_KEY`. To protect credentials:

- Back up the Portainer data volume securely
- Use tokens with minimal required scopes
- Rotate tokens periodically (create new token → update in Portainer → revoke old token)

## Auto-Update Stacks from Git

After setting up credentials, enable automatic stack updates:

1. In the stack settings, enable **Auto-update**.
2. Set the polling interval (e.g., every 5 minutes).

Portainer will pull the repository and redeploy if the compose file changes.
