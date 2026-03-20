# How to Deploy a Stack from a Template in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Templates, Stacks, DevOps

Description: Learn how to deploy multi-service applications from stack templates in Portainer with customizable variables.

## Introduction

Stack templates in Portainer combine the power of Docker Compose with the convenience of a template system. Instead of writing Compose files from scratch, you select a stack template, configure a few variables, and deploy a complete multi-service application. This guide shows you how.

## Prerequisites

- Portainer CE or BE installed
- Docker standalone or Docker Swarm environment
- Basic understanding of Docker Compose

## What Is a Stack Template

A stack template in Portainer (type `2`) contains:

```json
{
  "type": 2,
  "title": "WordPress",
  "description": "WordPress with MySQL database",
  "categories": ["CMS"],
  "platform": "linux",
  "logo": "https://portainer-io-assets.sfo2.digitaloceanspaces.com/logos/wordpress.png",
  "repository": {
    "url": "https://github.com/portainer/templates",
    "stackfile": "stacks/wordpress/docker-compose.yml"
  },
  "env": [
    {
      "name": "WORDPRESS_PORT",
      "label": "WordPress port",
      "default": "80"
    },
    {
      "name": "MYSQL_ROOT_PASSWORD",
      "label": "MySQL root password"
    }
  ]
}
```

## Step 1: Navigate to App Templates

1. Select your Docker environment in Portainer
2. Click **App Templates** in the sidebar
3. Browse or search for stack templates (look for the stack icon)

## Step 2: Find a Stack Template

Popular stack templates include:

- **WordPress** — WordPress + MySQL
- **Ghost** — Ghost CMS + MySQL
- **Nextcloud** — File sharing platform
- **Gitea** — Self-hosted Git service
- **MEAN** — MongoDB + Express + Angular + Node
- **Prometheus + Grafana** — Monitoring stack

## Step 3: Click the Stack Template

Click on your chosen template. The configuration panel expands showing:

- Template description
- Variable fields to fill in
- Optionally a preview of the Compose file

## Step 4: Configure Template Variables

For the **Ghost** stack template example:

```
Stack name:        my-ghost-blog

Ghost port:        2368
Ghost URL:         https://myblog.example.com
Ghost mail service: mailgun
Ghost mail user:   postmaster@myblog.example.com
Ghost mail password: [your-mailgun-password]

MySQL root password: [secure-root-password]
MySQL database:     ghost
MySQL user:         ghostuser
MySQL password:     [secure-ghost-password]
```

**Important:** Stack name must be unique and use only lowercase letters, numbers, and hyphens.

## Step 5: Review the Compose File (Optional)

Some template configurations show a preview of the resolved Compose file. Review it to understand what will be deployed:

```yaml
version: "3"
services:
  ghost:
    image: ghost:5-alpine
    ports:
      - "2368:2368"
    environment:
      database__client: mysql
      database__connection__host: db
      database__connection__user: ghostuser
      database__connection__password: "[your-password]"
      database__connection__database: ghost
      url: https://myblog.example.com
    volumes:
      - ghost-content:/var/lib/ghost/content
    depends_on:
      - db

  db:
    image: mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: "[root-password]"
      MYSQL_DATABASE: ghost
      MYSQL_USER: ghostuser
      MYSQL_PASSWORD: "[ghost-password]"
    volumes:
      - ghost-db:/var/lib/mysql

volumes:
  ghost-content:
  ghost-db:
```

## Step 6: Deploy the Stack

1. Click **Deploy the stack**
2. Portainer creates all services, volumes, and networks
3. Watch the deployment output for errors

## Step 7: Verify the Stack

1. Go to **Stacks** in the sidebar
2. Find your new stack (named `my-ghost-blog`)
3. Click it to see all containers
4. Verify all containers show **Running** status

Visit the application URL in your browser to confirm it works.

## Step 8: Manage the Deployed Stack

From the stack detail view:

- **Edit** — Modify the Compose file or environment variables
- **Stop** — Stop all services
- **Remove** — Delete the stack (optionally delete volumes)
- **Logs** — View per-container logs

## Updating a Stack from a Template

Stack templates create a standard Portainer stack. To update it:

1. Navigate to the stack in **Stacks**
2. Click **Editor** tab to modify the Compose file
3. Update image tags or configuration
4. Click **Update the stack**

## Creating a Custom Stack Template

Save any working Compose file as a reusable template:

1. Go to **App Templates > Custom Templates**
2. Click **+ Add custom template**
3. Choose **Stack** type
4. Paste your Compose file
5. Define variables using Mustache syntax: `{{ variable_name }}`

## Conclusion

Stack templates in Portainer combine the flexibility of Docker Compose with the simplicity of a template system. They are ideal for standardizing application deployments across teams and environments. Start with the built-in templates for common applications and build your own for internal services.
