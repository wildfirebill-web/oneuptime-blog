# How to Self-Host a Recipe Manager with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Mealie, Recipe Manager, Self-Hosting, Docker, Home Server

Description: Learn how to self-host Mealie, a modern recipe manager and meal planner, via Portainer with persistent storage and user authentication.

---

Mealie is a popular self-hosted recipe manager with a clean interface, meal planning, shopping lists, and API access. Portainer makes deploying and managing it simple.

## Compose Stack

```yaml
version: "3.8"

services:
  mealie:
    image: ghcr.io/mealie-recipes/mealie:latest
    restart: unless-stopped
    ports:
      - "9925:9000"
    environment:
      # Security settings
      ALLOW_SIGNUP: "true"         # Set to false after creating your account
      API_TOKENS_ENABLED: "true"
      # SMTP for email notifications (optional)
      SMTP_HOST: smtp.example.com
      SMTP_PORT: 587
      SMTP_AUTH_STRATEGY: TLS
      SMTP_FROM_EMAIL: noreply@example.com
      # Performance settings
      MAX_WORKERS: 2
      WEB_CONCURRENCY: 2
      # Timezone
      TZ: America/New_York
    volumes:
      - mealie_data:/app/data
```

The default credentials after first launch are:
- Email: `changeme@email.com`
- Password: `MyPassword`

Change these immediately after first login.

## Volumes and Data

```bash
# Mealie stores everything in /app/data:
# /app/data/recipes   - Recipe content and images
# /app/data/backups   - Automated backups
# /app/data/users     - User data
# /app/data/logs      - Application logs

# Back up the volume
docker run --rm -v mealie_data:/data alpine tar czf - /data > mealie-backup.tar.gz
```

## Importing Recipes

Mealie supports importing recipes from URLs automatically. Paste any recipe URL from the web and Mealie extracts ingredients, instructions, and images automatically.

You can also bulk import:

```bash
# Via the API
TOKEN="your-api-token"
curl -X POST "http://localhost:9925/api/recipes/create-url" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://www.foodnetwork.com/your-recipe"}'
```

## Meal Planning

Mealie includes a weekly meal planner. Navigate to **Meal Plans** in the sidebar to assign recipes to days of the week and automatically generate a shopping list.

## Monitoring

Use OneUptime to monitor `http://<host>:9925/api/app/about`. Mealie returns version and health information when running correctly. Alert on any downtime to keep your recipe collection accessible.
