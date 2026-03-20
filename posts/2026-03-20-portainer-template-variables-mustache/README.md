# How to Use Template Variables with Mustache Syntax in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Templates, Mustache, DevOps

Description: Learn how to use Mustache syntax to create dynamic, configurable template variables in Portainer custom templates.

## Introduction

Portainer's custom templates support Mustache-style variable syntax, enabling you to create parameterized templates that prompt users for configuration at deploy time. This guide covers the Mustache syntax supported by Portainer, best practices for template variables, and real-world examples.

## Prerequisites

- Portainer CE or BE with custom templates access
- Basic understanding of YAML and Docker Compose
- A template to parameterize

## Mustache Syntax Basics

Portainer uses a subset of Mustache/Go template syntax for variables:

```yaml
# Basic variable substitution

{{ .variable_name }}

# Variable with a default value
{{ .variable_name | default "fallback_value" }}

# Variable used inline in a string
  - APP_URL=https://{{ .domain }}/api
```

**Key rules:**
- Variable names must start with a dot: `.variable_name`
- Variable names are case-sensitive
- Names can only contain letters, numbers, and underscores
- Default values use the pipe `|` and `default` filter

## Basic Variable Examples

### Simple String Variable

```yaml
services:
  app:
    image: myapp:{{ .version }}
    # User must provide: version (e.g., "1.2.3" or "latest")
```

### Variable with Default

```yaml
services:
  web:
    ports:
      - "{{ .port | default "8080" }}:80"
    # Defaults to port 8080 if user doesn't provide a value
```

### Multiple Variables in One Line

```yaml
services:
  db:
    image: postgres:{{ .postgres_version | default "15" }}-{{ .postgres_variant | default "alpine" }}
    # Produces: postgres:15-alpine if defaults are used
```

## Common Variable Patterns

### Port Mapping

```yaml
services:
  frontend:
    ports:
      - "{{ .frontend_port | default "80" }}:80"
      - "{{ .frontend_ssl_port | default "443" }}:443"

  api:
    ports:
      - "{{ .api_port | default "3000" }}:3000"
```

### Environment Variables

```yaml
services:
  app:
    environment:
      # Required (no default)
      - DB_PASSWORD={{ .db_password }}
      - SECRET_KEY={{ .secret_key }}
      # Optional with defaults
      - LOG_LEVEL={{ .log_level | default "info" }}
      - MAX_CONNECTIONS={{ .max_connections | default "100" }}
      - DEBUG={{ .debug | default "false" }}
```

### Image Configuration

```yaml
services:
  app:
    image: {{ .registry | default "docker.io" }}/{{ .image_name }}:{{ .image_tag | default "latest" }}
```

### Volume Paths

```yaml
services:
  app:
    volumes:
      - {{ .data_dir | default "/var/lib/myapp" }}:/data
      - {{ .config_dir | default "/etc/myapp" }}:/config
```

Resource Limits

```yaml
services:
  app:
    deploy:
      resources:
        limits:
          memory: {{ .memory_limit | default "512m" }}
          cpus: "{{ .cpu_limit | default "0.5" }}"
```

## Complete Example Template

Here is a fully parameterized WordPress stack template:

```yaml
version: "3.8"

services:
  wordpress:
    image: wordpress:{{ .wordpress_version | default "latest" }}
    ports:
      - "{{ .wordpress_port | default "80" }}:80"
    environment:
      WORDPRESS_DB_HOST: database
      WORDPRESS_DB_USER: {{ .db_user | default "wordpress" }}
      WORDPRESS_DB_PASSWORD: {{ .db_password }}
      WORDPRESS_DB_NAME: {{ .db_name | default "wordpress" }}
      WORDPRESS_TABLE_PREFIX: {{ .table_prefix | default "wp_" }}
    volumes:
      - wordpress-data:/var/www/html
    depends_on:
      - database
    restart: unless-stopped

  database:
    image: mysql:{{ .mysql_version | default "8.0" }}
    environment:
      MYSQL_DATABASE: {{ .db_name | default "wordpress" }}
      MYSQL_USER: {{ .db_user | default "wordpress" }}
      MYSQL_PASSWORD: {{ .db_password }}
      MYSQL_ROOT_PASSWORD: {{ .db_root_password }}
    volumes:
      - mysql-data:/var/lib/mysql
    restart: unless-stopped

volumes:
  wordpress-data:
  mysql-data:
```

## Defining Variables in Portainer

For each Mustache variable, add a corresponding entry in the **Variables** section when creating the template:

```json
[
  {
    "name": "db_password",
    "label": "Database password",
    "description": "MySQL password for WordPress user (required)"
  },
  {
    "name": "db_root_password",
    "label": "MySQL root password",
    "description": "MySQL root user password (required)"
  },
  {
    "name": "wordpress_port",
    "label": "WordPress port",
    "description": "Host port for WordPress",
    "default": "80"
  },
  {
    "name": "db_name",
    "label": "Database name",
    "description": "Name of the WordPress database",
    "default": "wordpress"
  }
]
```

## Variable Validation Behavior

- Variables without a `default` value are treated as **required** in the Portainer UI
- The deploy button may be disabled until required variables are filled in
- Variables with defaults show the default as a placeholder in the input field

## Tips and Best Practices

1. **Always use defaults for optional settings** to make templates more user-friendly
2. **Never set defaults for passwords** - force users to provide their own
3. **Use descriptive labels** - "MySQL root password" is clearer than "root_pw"
4. **Avoid complex logic** - Portainer's Mustache support is limited; keep variables simple
5. **Test substitution** - mentally substitute test values to verify YAML remains valid

## Known Limitations

- No conditional blocks (`{{#if}}`) in Portainer's implementation
- No loops or iterations
- No arithmetic operations
- Complex Go template syntax may not be supported

Stick to `{{ .variable }}` and `{{ .variable | default "value" }}` for reliable results.

## Conclusion

Mustache variables are the key to creating flexible, reusable Portainer templates. By parameterizing ports, passwords, image versions, and configuration values, you create templates that work across environments without modification. Keep your variables simple, always provide defaults for optional settings, and clearly label required fields for the best user experience.
