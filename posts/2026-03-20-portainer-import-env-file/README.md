# How to Import Environment Variables from a .env File in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Containers, Configuration, DevOps

Description: Learn how to import environment variables into a Portainer container or stack from a .env file to simplify configuration management.

## Introduction

Managing environment variables one-by-one in the Portainer UI works fine for a few variables, but quickly becomes cumbersome when you have dozens. Portainer supports importing environment variables directly from a `.env` file, which is the standard format used by Docker Compose and most Twelve-Factor apps.

## Prerequisites

- Portainer installed with a connected Docker environment
- A prepared `.env` file with your environment variables

## Understanding .env File Format

A `.env` file is a simple text file with one `KEY=VALUE` pair per line:

```bash
# .env file format
# Lines starting with # are comments and are ignored
# Blank lines are ignored

# Application settings
APP_NAME=MyApplication
APP_ENV=production
APP_PORT=8080
APP_DEBUG=false

# Database settings
DB_HOST=postgres
DB_PORT=5432
DB_NAME=production_db
DB_USER=app_user
DB_PASSWORD=SecurePassword123!

# External services
REDIS_URL=redis://redis:6379/0
SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_USER=noreply@example.com
SMTP_PASSWORD=emailpassword

# Feature flags
FEATURE_NEW_UI=true
FEATURE_BETA_API=false

# Monitoring
LOG_LEVEL=warn
METRICS_ENABLED=true
SENTRY_DSN=https://abc123@sentry.io/12345
```

## Step 1: Prepare Your .env File

Create a `.env` file on your local machine following the format above.

Important rules:
- No spaces around the `=` sign.
- Values with spaces must be quoted: `APP_DESCRIPTION="My Great App"`.
- No trailing whitespace after values.
- Use `#` for comments.

```bash
# Example .env for a Node.js app
NODE_ENV=production
PORT=3000
DATABASE_URL=postgresql://user:password@db:5432/mydb
JWT_SECRET=your-secret-key-here
CORS_ORIGINS=https://app.example.com,https://api.example.com
MAX_UPLOAD_SIZE_MB=50
SESSION_TIMEOUT_MINUTES=60
```

## Step 2: Import .env in Portainer Container Creation

1. Navigate to **Containers > Add container**.
2. Set the container name and image.
3. Scroll to the **Env** tab.
4. Click **Load variables from .env file**.

A file picker dialog will appear. Select your `.env` file from your local machine.

Portainer will parse the file and populate the key-value fields with all variables from the file.

## Step 3: Review and Edit After Import

After importing, review the populated variables:
- Verify all expected variables are present.
- Edit any values that need to change for this specific environment.
- Add any additional variables not in the file.
- Remove any variables you don't need.

The import fills the form — you still have full control before deploying.

## Step 4: Using .env Files with Stacks

For Docker Compose stacks in Portainer, `.env` files work differently:

```yaml
# docker-compose.yml — references variables from .env
version: "3.8"

services:
  app:
    image: myorg/myapp:${APP_VERSION}
    environment:
      - NODE_ENV=${NODE_ENV}
      - DATABASE_URL=${DATABASE_URL}
      - JWT_SECRET=${JWT_SECRET}
    ports:
      - "${APP_PORT}:3000"

  db:
    image: postgres:${POSTGRES_VERSION:-15}
    environment:
      - POSTGRES_DB=${DB_NAME}
      - POSTGRES_USER=${DB_USER}
      - POSTGRES_PASSWORD=${DB_PASSWORD}
```

When creating a stack from the Portainer web editor:

1. Navigate to **Stacks > Add stack**.
2. Enter your compose YAML.
3. Under **Environment variables**, click **Load variables from .env file**.
4. Or manually add key-value pairs.

Variables are substituted when the stack is deployed.

## Managing .env Files Securely

Never commit `.env` files with sensitive values to version control:

```bash
# .gitignore
.env
.env.local
.env.production
*.env

# Commit a template without values:
.env.example
```

`.env.example` template:

```bash
# .env.example — commit this, not .env
APP_NAME=MyApplication
APP_ENV=production
APP_PORT=8080
DB_HOST=postgres
DB_PASSWORD=           # Set this in your deployment environment
JWT_SECRET=            # Generate with: openssl rand -hex 32
```

## Environment-Specific .env Files

Maintain separate `.env` files for each environment:

```
.env.development   → Local development
.env.staging       → Staging environment
.env.production    → Production (sensitive, keep secure!)
```

When deploying to different Portainer environments, import the appropriate `.env` file.

## Troubleshooting Import Issues

- **Variables not imported**: Ensure the file is plain text, not a Word document or binary.
- **Missing variables after import**: Check for lines with syntax errors (spaces around `=`).
- **Values truncated**: Values with special characters may need quoting.
- **Import button greyed out**: Check that you're in the Env tab of the container creation form.

## Conclusion

Importing `.env` files in Portainer saves time when deploying containers with many configuration variables. The import feature parses standard `.env` format, populates the form, and gives you a chance to review and edit before deploying. Combined with environment-specific `.env` files and proper secrets management, this approach keeps your container configuration clean, portable, and maintainable.
