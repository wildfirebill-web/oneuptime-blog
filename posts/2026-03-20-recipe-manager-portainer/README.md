# How to Self-Host a Recipe Manager with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Self-Hosted, Mealie, Recipes, Home Lab, Productivity

Description: Deploy Mealie as a self-hosted recipe manager with meal planning and shopping lists using Portainer.

## Introduction

Mealie is a self-hosted recipe manager that lets you store recipes, plan meals for the week, and generate shopping lists. You can import recipes from any website with a single URL, organize them by category, and share with family members. This guide covers deploying Mealie using Portainer.

## Prerequisites

- Portainer installed and running
- At least 512MB RAM
- A reverse proxy for external access (optional)

## Step 1: Deploy Mealie Stack

```yaml
# docker-compose.yml - Mealie Recipe Manager

version: "3.8"

networks:
  recipes_network:
    driver: bridge

volumes:
  mealie_data:
  mealie_db:

services:
  # PostgreSQL database (recommended for multi-user setups)
  mealie_db:
    image: postgres:15-alpine
    container_name: mealie_db
    restart: unless-stopped
    environment:
      - POSTGRES_DB=mealie
      - POSTGRES_USER=mealie
      - POSTGRES_PASSWORD=secure_db_password
    volumes:
      - mealie_db:/var/lib/postgresql/data
    networks:
      - recipes_network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U mealie"]
      interval: 10s
      retries: 5

  # Mealie application
  mealie:
    image: ghcr.io/mealie-recipes/mealie:latest
    container_name: mealie
    restart: unless-stopped
    depends_on:
      mealie_db:
        condition: service_healthy
    ports:
      - "9000:9000"
    environment:
      # Database connection
      - DB_ENGINE=postgres
      - POSTGRES_SERVER=mealie_db
      - POSTGRES_PORT=5432
      - POSTGRES_USER=mealie
      - POSTGRES_PASSWORD=secure_db_password
      - POSTGRES_DB=mealie

      # Application settings
      - BASE_URL=https://recipes.yourdomain.com
      - DEFAULT_GROUP=Home
      - DEFAULT_EMAIL=admin@yourdomain.com
      - DEFAULT_PASSWORD=change_this_password

      # Security settings
      - TOKEN_TIME=48
      - MAX_WORKERS=1
      - WEB_CONCURRENCY=1
      - ALLOW_SIGNUP=false

      # SMTP configuration
      - SMTP_HOST=smtp.gmail.com
      - SMTP_PORT=587
      - SMTP_FROM_EMAIL=noreply@yourdomain.com
      - SMTP_FROM_NAME=Mealie
      - SMTP_AUTH_STRATEGY=TLS
      - SMTP_USER=your-email@gmail.com
      - SMTP_PASSWORD=your-app-password

      # Timezone
      - TZ=America/New_York
    volumes:
      - mealie_data:/app/data
    networks:
      - recipes_network
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.mealie.rule=Host(`recipes.yourdomain.com`)"
      - "traefik.http.routers.mealie.entrypoints=websecure"
      - "traefik.http.routers.mealie.tls.certresolver=letsencrypt"
      - "traefik.http.services.mealie.loadbalancer.server.port=9000"
```

## Step 2: Simple Deployment (SQLite)

For a single-user setup, SQLite is sufficient:

```yaml
# docker-compose.yml - Mealie with SQLite (simple)
version: "3.8"

volumes:
  mealie_data:

services:
  mealie:
    image: ghcr.io/mealie-recipes/mealie:latest
    container_name: mealie
    restart: unless-stopped
    ports:
      - "9000:9000"
    environment:
      - ALLOW_SIGNUP=false
      - DEFAULT_EMAIL=admin@yourdomain.com
      - DEFAULT_PASSWORD=change_this_password
      - BASE_URL=http://your-server-ip:9000
      - TZ=America/New_York
    volumes:
      - mealie_data:/app/data
```

## Step 3: Import Recipes

### Import from URL

The easiest way to add recipes is by URL:

1. Click **Create Recipe** > **Import from URL**
2. Paste any recipe website URL (AllRecipes, Food Network, NYT Cooking, etc.)
3. Mealie automatically extracts the recipe using schema.org markup
4. Review and save

```bash
# Import via Mealie API
curl -X POST "https://recipes.yourdomain.com/api/recipes/create-url" \
  -H "Authorization: Bearer YOUR_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://www.allrecipes.com/recipe/12345/chicken-tikka-masala/"}'
```

### Import from Other Apps

```bash
# Import from other recipe apps (Nextcloud Cookbook, Paprika, etc.)
# Mealie accepts JSON-LD, YAML, and ZIP archives

# Export from Nextcloud Cookbook
docker exec nextcloud-cookbook php occ cookbook:export-all /tmp/recipes.zip

# Import to Mealie via API
curl -X POST "https://recipes.yourdomain.com/api/recipes/create-from-zip" \
  -H "Authorization: Bearer YOUR_API_TOKEN" \
  -F "archive=@/tmp/recipes.zip"
```

## Step 4: Set Up Meal Planning

### Weekly Meal Plan via API

```bash
# Add recipe to meal plan for next Monday
MONDAY=$(date -d "next monday" +%Y-%m-%d)

curl -X POST "https://recipes.yourdomain.com/api/groups/mealplans" \
  -H "Authorization: Bearer YOUR_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"date\": \"$MONDAY\",
    \"entryType\": \"dinner\",
    \"recipeId\": \"recipe-uuid-here\",
    \"title\": \"Chicken Tikka Masala\",
    \"text\": \"\"
  }"
```

### Generate Shopping List

```bash
# Create a shopping list from the week's meal plan
curl -X POST "https://recipes.yourdomain.com/api/groups/shopping/lists" \
  -H "Authorization: Bearer YOUR_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "Weekly Shopping"}'

# Add meal plan items to shopping list
curl -X POST "https://recipes.yourdomain.com/api/groups/shopping/lists/LIST_ID/recipe" \
  -H "Authorization: Bearer YOUR_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"recipeIncrements": [{"id": "recipe-uuid", "quantity": 1}]}'
```

## Step 5: Mobile App Setup

Mealie has an official companion app:

1. Install **Mealie** from App Store or Play Store
2. Enter your server URL
3. Log in with your credentials
4. Browse and plan meals on mobile

## Step 6: Configure Recipe Scrapers

Mealie can import from hundreds of recipe sites. Enable additional scrapers:

```bash
# Check which sites are supported
curl "https://recipes.yourdomain.com/api/app/about" | jq '.recipeScrapers'

# Import with custom scraper
curl -X POST "https://recipes.yourdomain.com/api/recipes/create-url/bulk" \
  -H "Authorization: Bearer YOUR_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "imports": [
      {"url": "https://www.seriouseats.com/recipe-url"},
      {"url": "https://www.bonappetit.com/another-recipe"}
    ]
  }'
```

## Step 7: Backup Your Recipe Collection

```bash
#!/bin/bash
# backup-mealie.sh
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/opt/backups/mealie"
mkdir -p "$BACKUP_DIR"

# Backup PostgreSQL
docker exec mealie_db pg_dump -U mealie mealie | \
  gzip > "$BACKUP_DIR/mealie_db_$DATE.sql.gz"

# Backup media files (recipe photos)
tar -czf "$BACKUP_DIR/mealie_data_$DATE.tar.gz" \
  $(docker volume inspect mealie_data -f '{{ .Mountpoint }}')

# Rotate (keep 30 days)
find "$BACKUP_DIR" -mtime +30 -delete
echo "Mealie backup: $DATE"
```

## Conclusion

You now have a beautiful self-hosted recipe manager running in Docker through Portainer. Mealie makes it easy to build and organize your recipe collection by importing from any website, plan your weekly meals, and auto-generate shopping lists. The mobile app keeps your recipe collection accessible in the kitchen. Use Portainer to keep Mealie updated and monitor its resource usage over time.
