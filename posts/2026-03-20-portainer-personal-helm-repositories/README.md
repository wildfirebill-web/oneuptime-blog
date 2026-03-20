# How to Configure Personal Helm Repositories in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Helm, Kubernetes, Repositories, Configuration, User Settings

Description: Learn how to add and manage personal Helm chart repositories in Portainer user settings for deploying applications to Kubernetes environments.

---

Portainer supports deploying applications to Kubernetes via Helm charts. By default, it includes public repositories like Bitnami and Artifact Hub. You can add private or custom Helm repositories in your personal user settings for access to internal charts.

## Adding a Personal Helm Repository

1. Log in to Portainer.
2. Click your username → **My Account**.
3. Scroll to **Helm repositories**.
4. Click **Add repository**.
5. Enter the repository URL.
6. Click **Save**.

The repository becomes available when deploying Helm charts in Kubernetes environments.

## Using a Public Repository

```bash
# Example: add the Jetstack repository for cert-manager

# Repository URL: https://charts.jetstack.io

# Or the Ingress-NGINX repository
# Repository URL: https://kubernetes.github.io/ingress-nginx

# These work without credentials
```

## Adding a Private Helm Repository

For private repositories (like a self-hosted Chartmuseum or Nexus), you can add basic authentication:

```bash
# If your private Helm repo uses HTTP Basic Auth, the URL format is:
# https://user:password@helm.internal.example.com/

# Or add the credentials via URL:
https://myuser:mypassword@charts.example.com/stable
```

Note: Store the credentials-embedded URL securely - it is stored in Portainer's user settings.

## Setting Up Chartmuseum for Private Charts

Deploy a private Helm repository with Portainer:

```yaml
version: "3.8"
services:
  chartmuseum:
    image: ghcr.io/helm/chartmuseum:latest
    restart: unless-stopped
    ports:
      - "8088:8080"
    environment:
      STORAGE: local
      STORAGE_LOCAL_ROOTDIR: /charts
      DISABLE_API: "false"
    volumes:
      - chartmuseum_data:/charts

volumes:
  chartmuseum_data:
```

Add charts by pushing to Chartmuseum:

```bash
# Push a chart to your Chartmuseum
helm plugin install https://github.com/chartmuseum/helm-push
helm cm-push ./mychart/ http://localhost:8088
```

Then add `http://chartmuseum:8088` as a Helm repository in Portainer.

## Repository Scope: Personal vs Global

Personal repositories (in My Account) are only available to the logged-in user. Admins can add global repositories via **Settings > General** that are available to all users.
