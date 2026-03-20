# Best Practices for Template Management in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Templates, Docker, Best Practices, Self-Service, DevOps, App Templates

Description: Build and manage application templates in Portainer to enable developer self-service, standardize deployments, and reduce configuration errors across your organization.

---

Portainer templates (App Templates) allow teams to define pre-configured applications that anyone can deploy with a few clicks. Well-designed templates reduce deployment errors, enforce standards, and enable developer self-service.

## Types of Portainer Templates

- **Container templates** — single container deployments with pre-set configuration
- **Stack templates** — Docker Compose-based multi-service deployments
- **Edge stack templates** — Compose stacks for Edge Agent deployments

## Building a Custom Template Library

Create a `templates.json` file and host it on a web server or Git repository:

```json
{
  "version": "3",
  "templates": [
    {
      "type": 1,
      "title": "PostgreSQL Database",
      "description": "Standard PostgreSQL 16 database with persistent storage",
      "categories": ["database"],
      "platform": "linux",
      "logo": "https://upload.wikimedia.org/wikipedia/commons/2/29/Postgresql_elephant.svg",
      "image": "postgres:16-alpine",
      "env": [
        {
          "name": "POSTGRES_DB",
          "label": "Database Name",
          "description": "Name of the initial database to create"
        },
        {
          "name": "POSTGRES_USER",
          "label": "Database User",
          "description": "Username for the database"
        },
        {
          "name": "POSTGRES_PASSWORD",
          "label": "Database Password",
          "description": "Secure password for the database user",
          "preset": false
        }
      ],
      "volumes": [
        {
          "container": "/var/lib/postgresql/data",
          "description": "PostgreSQL data directory"
        }
      ],
      "restart_policy": "unless-stopped",
      "ports": ["5432:5432/tcp"]
    },
    {
      "type": 2,
      "title": "WordPress + MySQL",
      "description": "WordPress CMS with MySQL database backend",
      "categories": ["cms", "blog"],
      "platform": "linux",
      "logo": "https://upload.wikimedia.org/wikipedia/commons/9/98/WordPress_blue_logo.svg",
      "repository": {
        "url": "https://github.com/portainer/templates",
        "stackfile": "stacks/wordpress/docker-compose.yml"
      },
      "env": [
        {
          "name": "MYSQL_ROOT_PASSWORD",
          "label": "MySQL Root Password",
          "preset": false
        },
        {
          "name": "WORDPRESS_ADMIN_EMAIL",
          "label": "Admin Email"
        }
      ]
    }
  ]
}
```

## Configuring Custom Templates in Portainer

1. Host your `templates.json` on a web server or GitHub raw URL
2. In Portainer, go to **Settings > App Templates**
3. Set **URL**: `https://raw.githubusercontent.com/yourorg/portainer-templates/main/templates.json`

## Template Design Principles

**Make required fields visible:**

```json
{
  "name": "ADMIN_PASSWORD",
  "label": "Admin Password",
  "description": "At least 16 characters",
  "preset": false    // User must set this value — no default
}
```

**Pre-set sensible defaults:**

```json
{
  "name": "MAX_CONNECTIONS",
  "label": "Max Database Connections",
  "description": "Default is 100, adjust based on expected load",
  "default": "100"
}
```

**Use categories for discoverability:**

Common categories: `database`, `cms`, `monitoring`, `messaging`, `storage`, `security`, `networking`

## Version Control Templates

Store your template library in Git:

```
portainer-templates/
├── templates.json            # Template index
├── stacks/
│   ├── wordpress/
│   │   ├── docker-compose.yml
│   │   └── README.md
│   ├── monitoring/
│   │   ├── docker-compose.yml
│   │   └── README.md
│   └── nextcloud/
│       ├── docker-compose.yml
│       └── README.md
└── CHANGELOG.md
```

Use pull requests for template updates — changes go through review before reaching users.

## Testing Templates

Before publishing templates:

1. Deploy the template in a test environment
2. Verify all environment variables are passed correctly
3. Test that persistent volumes work on container restart
4. Confirm health checks pass after startup

## Summary

Well-managed Portainer templates accelerate deployments, enforce standards, and reduce configuration errors. Keep templates in version control, make required fields explicit, use categories for discoverability, and test templates before publishing. A good template library is a force multiplier for developer productivity.
