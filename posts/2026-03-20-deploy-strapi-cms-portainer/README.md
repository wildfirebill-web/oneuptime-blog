# How to Deploy Strapi CMS via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Strapi, CMS, Portainer, Docker, Headless CMS, Content Management, Node.js

Description: Deploy Strapi headless CMS with PostgreSQL database via Portainer for a production-ready content management platform with a REST and GraphQL API for your frontend applications.

---

Strapi is the leading open-source headless CMS. It provides a customizable admin panel and auto-generated REST and GraphQL APIs for your content. Portainer makes it easy to deploy and manage Strapi alongside its database.

## Deploy Strapi Stack

```yaml
# strapi-stack.yml

version: "3.8"
services:
  strapi:
    image: strapi/strapi:latest
    environment:
      DATABASE_CLIENT: postgres
      DATABASE_HOST: postgres
      DATABASE_PORT: 5432
      DATABASE_NAME: strapi
      DATABASE_USERNAME: strapi
      DATABASE_PASSWORD: ${DATABASE_PASSWORD:-strapi_password}
      JWT_SECRET: ${JWT_SECRET:-your-jwt-secret-change-this}
      APP_KEYS: ${APP_KEYS:-key1,key2,key3,key4}
      API_TOKEN_SALT: ${API_TOKEN_SALT:-your-salt}
      ADMIN_JWT_SECRET: ${ADMIN_JWT_SECRET:-your-admin-jwt}
      NODE_ENV: production
    volumes:
      - strapi-uploads:/opt/app/public/uploads
    ports:
      - "1337:1337"
    depends_on:
      postgres:
        condition: service_healthy
    restart: unless-stopped
    networks:
      - strapi-net

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: strapi
      POSTGRES_USER: strapi
      POSTGRES_PASSWORD: ${DATABASE_PASSWORD:-strapi_password}
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U strapi"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped
    networks:
      - strapi-net

volumes:
  strapi-uploads:
  postgres-data:

networks:
  strapi-net:
    driver: bridge
```

## Environment Variables in Portainer

Set these in Portainer's stack environment variables for security:

```text
DATABASE_PASSWORD=<strong-password>
JWT_SECRET=<32-character-random-string>
APP_KEYS=<key1>,<key2>,<key3>,<key4>
API_TOKEN_SALT=<random-salt>
ADMIN_JWT_SECRET=<admin-jwt-secret>
```

Generate secrets with:

```bash
openssl rand -base64 32   # For JWT_SECRET and ADMIN_JWT_SECRET
openssl rand -base64 16   # For API_TOKEN_SALT
```

## Accessing the Admin Panel

After deployment, navigate to `http://host:1337/admin` to:

1. Create your first admin user
2. Define content types (blog posts, products, pages, etc.)
3. Add content and manage media uploads
4. Configure API tokens for frontend access

## Using the Strapi API

Strapi auto-generates REST and GraphQL APIs for your content:

```javascript
// Fetch blog posts from your Strapi API
const posts = await fetch(
  "http://strapi-host:1337/api/blog-posts?populate=*",
  {
    headers: {
      Authorization: `Bearer ${process.env.STRAPI_API_TOKEN}`
    }
  }
).then(r => r.json());
```

## Nginx Reverse Proxy

Put Strapi behind Nginx for production:

```yaml
  nginx:
    image: nginx:1.25-alpine
    volumes:
      - /opt/nginx/strapi.conf:/etc/nginx/conf.d/strapi.conf:ro
    ports:
      - "443:443"
    depends_on:
      - strapi
```

```nginx
server {
    listen 443 ssl;
    server_name cms.example.com;
    
    location / {
        proxy_pass http://strapi:1337;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_cache_bypass $http_upgrade;
    }
    
    # Serve uploads directly from Nginx for better performance
    location /uploads {
        proxy_pass http://strapi:1337/uploads;
    }
}
```

## Backup Strapi Data

Back up both the database and the uploads volume:

```bash
# Backup PostgreSQL
docker exec strapi_postgres_1 pg_dump -U strapi strapi > strapi-backup-$(date +%Y%m%d).sql

# Backup uploads volume
docker run --rm \
  -v strapi-uploads:/uploads:ro \
  -v /opt/backups:/backups \
  alpine tar czf "/backups/strapi-uploads-$(date +%Y%m%d).tar.gz" /uploads
```

## Summary

Strapi deployed via Portainer provides a production-ready headless CMS with auto-generated REST and GraphQL APIs. The Portainer stack makes it easy to manage Strapi alongside its PostgreSQL database, and environment variables keep secrets out of your compose file.
