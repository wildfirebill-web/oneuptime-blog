# How to Create Custom Templates from a Git Repository in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Templates, Git, DevOps

Description: Learn how to create versioned Portainer custom templates by linking to Compose files stored in a Git repository.

## Introduction

Storing Portainer templates in a Git repository provides version control, collaboration, and consistency across multiple Portainer instances. Instead of manually maintaining templates in each Portainer installation, you maintain a single source of truth in Git and link Portainer to it. This guide covers creating custom templates from a Git repository.

## Prerequisites

- Portainer CE or BE 2.x+
- A Git repository with Docker Compose files (GitHub, GitLab, Gitea, Bitbucket)
- Portainer network access to the Git host
- (For private repos) A personal access token with read access

## Repository Structure

Organize your template repository clearly:

```text
portainer-templates/
├── README.md
├── stacks/
│   ├── monitoring/
│   │   ├── docker-compose.yml    # Prometheus + Grafana stack
│   │   └── README.md
│   ├── wordpress/
│   │   ├── docker-compose.yml
│   │   └── README.md
│   └── gitea/
│       ├── docker-compose.yml
│       └── README.md
└── containers/
    ├── nginx/
    │   └── docker-compose.yml
    └── postgres/
        └── docker-compose.yml
```

## Step 1: Prepare Your Compose File in Git

Create a Compose file with Mustache variables where customization is needed:

```yaml
# stacks/monitoring/docker-compose.yml

services:
  prometheus:
    image: prom/prometheus:{{ .prometheus_tag | default "latest" }}
    ports:
      - "{{ .prometheus_port | default "9090" }}:9090"
    volumes:
      - prometheus-data:/prometheus
    restart: unless-stopped

  grafana:
    image: grafana/grafana:{{ .grafana_tag | default "latest" }}
    ports:
      - "{{ .grafana_port | default "3000" }}:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD={{ .admin_password }}
      - GF_SERVER_ROOT_URL=https://{{ .domain }}/grafana
    volumes:
      - grafana-data:/var/lib/grafana
    depends_on:
      - prometheus
    restart: unless-stopped

volumes:
  prometheus-data:
  grafana-data:
```

Commit and push this file to your repository.

## Step 2: Create the Custom Template in Portainer

1. Go to **App Templates > Custom templates**
2. Click **+ Add custom template**
3. Select **Repository** as the build method

## Step 3: Fill in Repository Details

| Field | Value |
|-------|-------|
| Repository URL | `https://github.com/myorg/portainer-templates` |
| Repository reference | `refs/heads/main` |
| Compose file path | `stacks/monitoring/docker-compose.yml` |
| Authentication | Toggle ON for private repos |
| Username | Your Git username (if private) |
| Personal access token | Your PAT (if private) |

For GitHub personal access tokens, create one at **Settings → Developer settings → Personal access tokens** with `repo:read` scope.

## Step 4: Add Template Metadata

```text
Title:       Monitoring Stack (Prometheus + Grafana)
Description: Production-ready monitoring with alerting

Categories:  monitoring, observability
Platform:    linux
Type:        Stack
```

## Step 5: Configure Template Variables

Add variable definitions matching the Mustache variables in your Compose file:

```text
Variable 1:
  Name:        admin_password
  Label:       Grafana admin password
  Description: Password for Grafana admin account
  Default:     (empty - required)

Variable 2:
  Name:        domain
  Label:       Application domain
  Description: Domain name for Grafana URL configuration
  Default:     localhost

Variable 3:
  Name:        grafana_port
  Label:       Grafana UI port
  Description: Host port to expose Grafana
  Default:     3000

Variable 4:
  Name:        prometheus_port
  Label:       Prometheus UI port
  Description: Host port to expose Prometheus
  Default:     9090
```

## Step 6: Save the Template

Click **Create custom template**. Portainer fetches the Compose file from the repository to validate it, then saves the template.

## Updating Templates from Git

When you update the Compose file in your repository, users get the latest version next time they deploy the template. Portainer fetches the file fresh from the repository each time a template is deployed.

To update the template configuration itself (variables, metadata):

1. Go to **Custom templates**
2. Click **Edit** on the template
3. Update the repository URL, branch, path, or variable definitions
4. Save

## Using Tags for Stable Template Versions

Point to specific tags for stable releases:

```text
Repository reference: refs/tags/v2.0.0   # Always deploy from tag v2.0.0
```

Or branches for environment-specific templates:

```text
refs/heads/production    # Production template branch
refs/heads/staging       # Staging template branch
```

## Private Repository Authentication

### GitHub Personal Access Token

```bash
# Create a fine-grained PAT with:

# Repository: read access to Contents
# No other permissions needed
```

### GitLab Deploy Token

```bash
# Create at: Repository Settings → Repository → Deploy tokens
# Scope: read_repository
```

### Self-hosted Gitea

```bash
# Create at: User Settings → Applications → Generate Token
# No special scope needed for public repos
```

## Multiple Portainer Instances

Git-based templates shine when managing multiple Portainer instances. Since templates are stored centrally in Git, all instances can reference the same repository and always get the latest templates without manual synchronization.

## Conclusion

Git-backed custom templates are the recommended approach for production Portainer environments. They provide version control for your templates, enable team collaboration, and make it easy to maintain consistency across multiple Portainer instances. Set up a dedicated template repository and structure it to mirror your application catalog.
