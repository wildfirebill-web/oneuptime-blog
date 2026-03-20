# How to Create Custom Templates in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Templates, Custom Templates, DevOps

Description: Learn the different ways to create custom templates in Portainer for reusable container and stack deployments.

## Introduction

Portainer's Custom Templates feature lets you build your own catalog of reusable deployment configurations. Whether you have a standard application stack, an internal tool, or a frequently-used container configuration, custom templates save time and ensure consistency across deployments. This guide provides an overview of all custom template creation methods.

## Prerequisites

- Portainer CE or BE installed
- Admin or team-level access to manage templates
- Basic understanding of Docker Compose

## Custom Template Types

Portainer supports two custom template types:

| Type | Description | Use Case |
|------|-------------|----------|
| **Container** | Single-container deployment | Simple services, utilities |
| **Stack** | Multi-service Docker Compose | Full applications |

## Creation Methods Overview

Portainer provides three ways to create custom templates:

1. **Web editor** - Write or paste the template directly in the browser
2. **Git repository** - Point to a Compose file in a Git repo
3. **File upload** - Upload a local Compose file

## Step 1: Navigate to Custom Templates

1. Select your environment in Portainer
2. Click **App Templates** in the sidebar
3. Click the **Custom templates** tab
4. Click **+ Add custom template**

## Step 2: Fill in Template Metadata

Regardless of creation method, all templates require:

```text
Title:       My Application Stack     # Display name in catalog
Description: Deploys my app with...   # Short description

Platform:    Linux                    # linux or windows
Type:        Stack                    # Container or Stack
Logo URL:    https://...              # Optional logo image URL
```

**Categories** (optional) help organize your template catalog:
```text
Categories: webserver, monitoring, database
```

## Step 3: Choose a Creation Method

### Method 1: Web Editor

Write your Compose file directly in the browser. Best for quick templates or testing.

- Paste an existing Compose file
- Use the built-in YAML editor with syntax highlighting
- Add Mustache variables for user inputs: `{{ .variable_name }}`

### Method 2: Git Repository

Link to a Compose file in a Git repository. Best for version-controlled templates.

- Enter the repository URL
- Set the branch (e.g., `main`)
- Specify the path to the Compose file within the repo
- Add credentials if the repo is private

### Method 3: File Upload

Upload a Compose file from your local machine. Best for one-time imports.

- Click **Upload**
- Select your `docker-compose.yml` file
- The file contents load into the editor for review

## Step 4: Add Template Variables

Variables let template users customize deployments. Use Mustache syntax:

```yaml
# In the Compose file, use variables like:

services:
  app:
    image: {{ .image_name }}:{{ .image_tag }}
    ports:
      - "{{ .app_port }}:8080"
    environment:
      - DB_PASSWORD={{ .db_password }}
      - APP_SECRET={{ .app_secret }}
```

For each variable, define it in the **Variables** section:

```json
{
  "name": "app_port",
  "label": "Application port",
  "description": "Host port to expose the application on",
  "default": "8080"
}
```

## Step 5: Save the Template

1. Review all fields
2. Click **Create custom template**
3. The template appears in the **Custom templates** catalog

## Step 6: Use the Template

1. Go to **App Templates > Custom templates**
2. Find your template
3. Click it to expand the configuration panel
4. Fill in variable values
5. Click **Deploy**

## Managing Custom Templates

### Edit a Template

1. Click the pencil icon on the template card
2. Modify the Compose file or metadata
3. Save changes

Note: Editing a template does not affect already-deployed stacks.

### Duplicate a Template

Use the duplicate option to create variants (e.g., dev vs. production versions of the same template).

### Delete a Template

Click the trash icon on the template card. Deployed stacks are not affected.

## Template Best Practices

```yaml
# Good template: parameterize changeable values
services:
  web:
    image: {{ .image }}:{{ .tag | default "latest" }}
    ports:
      - "{{ .port | default "80" }}:80"
    environment:
      - DB_HOST={{ .db_host | default "database" }}
      - DB_PASSWORD={{ .db_password }}   # Required, no default
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: {{ .memory_limit | default "512m" }}
```

- Always provide sensible defaults for optional variables
- Make passwords and secrets required (no default)
- Document variables with clear labels and descriptions

## Sharing Templates Across Teams

In Portainer Business Edition, templates can be shared with specific teams or made globally available to all users in an environment.

## Conclusion

Custom templates in Portainer are a powerful way to standardize and speed up application deployments. By creating reusable templates for your most common workloads, you reduce configuration errors and make it easy for team members to deploy services correctly every time. Choose the creation method that fits your workflow - web editor for quick iteration, Git for versioned templates, or file upload for one-time imports.
