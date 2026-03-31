# How to Create a Stack from a Custom Template in Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Stack, Template, DevOps

Description: Learn how to create reusable custom app templates in Portainer that teams can deploy with one click and customizable parameters.

## Introduction

Portainer's App Templates let you define reusable Docker Compose configurations that team members can deploy with a single click, filling in only the values that change between deployments. Custom templates are ideal for standardizing how your team deploys common services - databases, monitoring stacks, web applications - ensuring consistency and reducing the chance of misconfiguration. Portainer BE (Business Edition) also supports custom template categories and visibility controls.

## Prerequisites

- Portainer installed (CE or BE)
- Admin or appropriate role to create templates
- A Docker Compose YAML to use as the template

## Step 1: Navigate to App Templates

1. Log into Portainer.
2. Navigate to **App Templates** in the left menu.
3. Click **Add Custom Template**.

## Step 2: Define the Template Metadata

Fill in the template details:

```text
Title:       WordPress with MySQL
Description: Deploys WordPress CMS with a MySQL 8 database and persistent volumes

Note:        Change DB_PASSWORD before deploying to production
Logo:        https://cdn.example.com/wordpress-logo.png  (optional)
Platform:    linux
Type:        Standalone (or Swarm)
```

## Step 3: Write the Template Compose Content

In the **Web editor** tab, enter the Compose YAML:

```yaml
# Custom template: WordPress + MySQL

version: "3.8"

services:
  wordpress:
    image: wordpress:{{ .Values.wordpressVersion | default "6-apache" }}
    restart: unless-stopped
    ports:
      - "{{ .Values.port | default 8080 }}:80"
    networks:
      - wp-net
    environment:
      - WORDPRESS_DB_HOST=mysql
      - WORDPRESS_DB_NAME={{ .Values.dbName | default "wordpress" }}
      - WORDPRESS_DB_USER={{ .Values.dbUser | default "wpuser" }}
      - WORDPRESS_DB_PASSWORD={{ .Values.dbPassword }}
    volumes:
      - wp_content:/var/www/html/wp-content

  mysql:
    image: mysql:8-oracle
    restart: unless-stopped
    networks:
      - wp-net
    environment:
      - MYSQL_DATABASE={{ .Values.dbName | default "wordpress" }}
      - MYSQL_USER={{ .Values.dbUser | default "wpuser" }}
      - MYSQL_PASSWORD={{ .Values.dbPassword }}
      - MYSQL_ROOT_PASSWORD={{ .Values.dbRootPassword }}
    volumes:
      - mysql_data:/var/lib/mysql

networks:
  wp-net:
    driver: bridge

volumes:
  wp_content:
  mysql_data:
```

Note: Standard Portainer templates use environment variables rather than Go template syntax. The actual template variables are defined in the **Environment variables** section:

```yaml
# Simpler template without Go syntax:
services:
  wordpress:
    image: wordpress:latest
    ports:
      - "${WP_PORT}:80"
    environment:
      - WORDPRESS_DB_PASSWORD=${DB_PASSWORD}
```

## Step 4: Add Template Variables (Environment Variables)

In the **Environment variables** section, define variables that users will fill in:

| Name | Label | Default | Description |
|------|-------|---------|-------------|
| `WP_PORT` | WordPress Port | `8080` | Host port for WordPress |
| `DB_PASSWORD` | Database Password | _(empty)_ | MySQL password |
| `DB_ROOT_PASSWORD` | DB Root Password | _(empty)_ | MySQL root password |

## Step 5: Save and Deploy from Template

1. Click **Create custom template**.
2. The template appears in the **App Templates** → **Custom Templates** section.

To deploy from the template:
1. Navigate to **App Templates**.
2. Click the template card.
3. Fill in the variable values.
4. Enter a stack name.
5. Click **Deploy the stack**.

## Step 6: Share Templates via URL (Portainer BE)

Portainer BE supports loading templates from an external JSON file:

```json
[
  {
    "type": 3,
    "title": "WordPress with MySQL",
    "description": "WordPress CMS with MySQL 8 database",
    "categories": ["CMS", "Database"],
    "platform": "linux",
    "logo": "https://cdn.example.com/wordpress.png",
    "repository": {
      "url": "https://github.com/myorg/portainer-templates",
      "stackfile": "wordpress/docker-compose.yml"
    },
    "env": [
      {
        "name": "WP_PORT",
        "label": "WordPress Port",
        "default": "8080"
      },
      {
        "name": "DB_PASSWORD",
        "label": "Database Password"
      }
    ]
  }
]
```

Configure the template URL in Portainer:
1. Navigate to **Settings** → **App Templates**.
2. Set the URL to your JSON template file.
3. Save. Templates load from the URL.

## Step 7: Organize Templates by Category

Use categories to group related templates:

```json
"categories": ["Database", "Storage"]         // Database templates
"categories": ["Monitoring", "Observability"]  // Monitoring templates
"categories": ["Web", "Proxy"]                 // Web server templates
"categories": ["Development", "Tools"]         // Dev tools
```

## Conclusion

Custom App Templates in Portainer enable self-service deployments while maintaining standards. Define the Compose YAML once, expose the values that vary (passwords, ports, image tags) as environment variable fields, and team members can deploy correctly configured stacks without knowing Docker Compose syntax. For organizations with multiple environments, host templates in a Git repository and configure Portainer to load from that URL - template updates flow to all Portainer instances automatically.
