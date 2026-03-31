# How to Create Custom Templates via File Upload in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Template, DevOps

Description: Learn how to create Portainer custom templates by uploading Docker Compose files directly from your local machine.

## Introduction

File upload is the simplest way to create a Portainer custom template when you have a Docker Compose file on your local machine. It is ideal for one-off templates, migrating existing Compose files into Portainer's template catalog, or quickly sharing configurations without setting up a Git repository. This guide walks through the process.

## Prerequisites

- Portainer CE or BE installed
- A valid Docker Compose file on your local machine
- Admin or environment-admin access in Portainer

## Step 1: Prepare Your Compose File

Before uploading, ensure your Compose file is ready for templating. Replace hardcoded values with Mustache variables for anything that should be configurable:

```yaml
# Before: hardcoded values

services:
  app:
    image: myapp:1.2.3
    ports:
      - "8080:8080"
    environment:
      - DB_PASSWORD=supersecret123
      - APP_DOMAIN=myapp.example.com

# After: parameterized with Mustache variables
services:
  app:
    image: myapp:{{ .version | default "latest" }}
    ports:
      - "{{ .port | default "8080" }}:8080"
    environment:
      - DB_PASSWORD={{ .db_password }}
      - APP_DOMAIN={{ .domain }}
```

Save this as a `.yml` file locally (e.g., `myapp-template.yml`).

## Step 2: Navigate to Custom Template Creation

1. Open Portainer and select your environment
2. Click **App Templates > Custom templates**
3. Click **+ Add custom template**
4. Select **Upload** as the build method

## Step 3: Fill in Template Metadata

```text
Title:       My Application
Description: Deploys My Application with configurable settings

Categories:  application, internal
Platform:    linux
Type:        Stack
Logo URL:    https://mycompany.com/logo.png  (optional)
```

## Step 4: Upload the Compose File

1. Click the **Upload** button or drag-and-drop your file onto the upload area
2. Select your `myapp-template.yml` file
3. The file contents load into the editor for review

You can review and edit the content after upload in the web editor before saving.

## Step 5: Review the Loaded Content

After upload, the editor shows your Compose file:

```yaml
# Review and verify the uploaded content
services:
  app:
    image: myapp:{{ .version | default "latest" }}
    ports:
      - "{{ .port | default "8080" }}:8080"
    environment:
      - DB_PASSWORD={{ .db_password }}
      - APP_DOMAIN={{ .domain }}
    volumes:
      - app-data:/data
    restart: unless-stopped

volumes:
  app-data:
```

Make any corrections directly in the editor.

## Step 6: Add Variable Definitions

Click **Add variable** for each Mustache variable in your Compose file:

### version variable

```bash
Name:        version
Label:       Application version
Description: Docker image tag to deploy
Default:     latest
```

### port variable

```text
Name:        port
Label:       Application port
Description: Host port to expose the application
Default:     8080
```

### db_password variable

```text
Name:        db_password
Label:       Database password
Description: Password for the application database (required)
Default:     (leave empty to make required)
```

### domain variable

```text
Name:        domain
Label:       Application domain
Description: Domain name for the application
Default:     localhost
```

## Step 7: Create the Template

1. Verify all metadata, the Compose file content, and variables
2. Click **Create custom template**
3. The template is saved to Portainer's database

## Step 8: Verify and Test

1. Go to **App Templates > Custom templates**
2. Find your uploaded template
3. Click it to see the variable configuration form
4. Fill in test values and click **Deploy the stack**
5. Verify the deployment succeeds

## Real-World Example: Uploading a Complete Application Stack

Here is an example of a production-ready Compose file to upload and convert to a template:

```yaml
# webapp-template.yml
version: "3.8"

services:
  frontend:
    image: nginx:{{ .nginx_version | default "alpine" }}
    ports:
      - "{{ .frontend_port | default "80" }}:80"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - backend
    restart: unless-stopped

  backend:
    image: {{ .backend_image }}:{{ .backend_tag | default "latest" }}
    environment:
      - DATABASE_URL=postgresql://{{ .db_user }}:{{ .db_password }}@postgres:5432/{{ .db_name }}
      - SECRET_KEY={{ .secret_key }}
      - DEBUG={{ .debug | default "false" }}
    depends_on:
      - postgres
    restart: unless-stopped

  postgres:
    image: postgres:{{ .postgres_version | default "15-alpine" }}
    environment:
      - POSTGRES_DB={{ .db_name }}
      - POSTGRES_USER={{ .db_user }}
      - POSTGRES_PASSWORD={{ .db_password }}
    volumes:
      - postgres-data:/var/lib/postgresql/data
    restart: unless-stopped

volumes:
  postgres-data:
```

## Limitations of File Upload Method

- No automatic sync with the source file (changes must be re-uploaded or edited in the web editor)
- No version history within Portainer
- For frequently updated templates, prefer the Git repository method

## Migration from Existing Compose Files

The file upload method is perfect for migrating existing Docker Compose deployments into Portainer templates:

1. Locate your existing `docker-compose.yml`
2. Replace environment-specific values with Mustache variables
3. Upload to Portainer as a custom template
4. Delete the old hardcoded Compose file

## Conclusion

File upload template creation is the quickest way to get a local Compose file into Portainer's template catalog. It works well for one-time imports and small teams. For ongoing template maintenance and multi-instance deployments, consider migrating to Git-backed templates for version control and easier updates.
