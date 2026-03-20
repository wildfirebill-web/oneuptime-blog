# How to Set Up a Staging Environment with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Staging, Environment Management, Docker, CI/CD, Testing

Description: Learn how to set up and manage a staging environment with Portainer using separate stacks, environment-specific configurations, and data isolation.

---

A staging environment mirrors production but is used for final testing before releases. Portainer makes it easy to manage staging and production on the same host or separate hosts by using distinct stacks and environment variables.

## Architecture: Staging on the Same Host

Running staging alongside production on the same Portainer instance saves infrastructure costs:

```mermaid
graph TD
    Portainer --> ProdStack[Production Stack :80/:443]
    Portainer --> StagingStack[Staging Stack :8080/:8443]
    ProdStack --> ProdDB[(Production DB)]
    StagingStack --> StagingDB[(Staging DB)]
```

## Creating a Staging Stack

Duplicate your production compose file and adjust ports and volume names:

```yaml
version: "3.8"
# docker-compose.staging.yml

services:
  api:
    image: myregistry.example.com/my-app:staging
    environment:
      NODE_ENV: staging
      DATABASE_URL: "postgresql://appuser:stagingpassword@postgres-staging:5432/appdb_staging"
      LOG_LEVEL: debug
    ports:
      - "8080:3000"    # Different external port from production
    networks:
      - staging_net

  postgres-staging:
    image: postgres:15
    environment:
      POSTGRES_DB: appdb_staging
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: stagingpassword
    volumes:
      - postgres_staging_data:/var/lib/postgresql/data   # Separate volume
    networks:
      - staging_net

volumes:
  postgres_staging_data:    # Isolated from production volumes

networks:
  staging_net:
    driver: bridge
```

## Environment Variable Management

Use Portainer's stack environment variables to toggle staging-specific settings without changing the compose file:

| Variable | Production | Staging |
|----------|------------|---------|
| `NODE_ENV` | production | staging |
| `LOG_LEVEL` | warn | debug |
| `RATE_LIMIT` | 100 | 1000 |
| `CACHE_TTL` | 3600 | 60 |
| `DEBUG_MODE` | false | true |

Set these in **Stacks > [staging stack] > Environment variables** in Portainer.

## Seeding Staging with Production Data

Copy a sanitized snapshot of production data to staging periodically:

```bash
#!/bin/bash
# sanitize-and-seed-staging.sh

# Dump production database
docker exec $(docker ps -qf name=postgres_prod) \
  pg_dump -U appuser appdb > /tmp/prod-dump.sql

# Anonymize sensitive data
sed -i "s/[a-zA-Z0-9._%+-]*@[a-zA-Z0-9.-]*\.[a-zA-Z]*/'user@example.com'/g" /tmp/prod-dump.sql

# Load into staging
docker exec -i $(docker ps -qf name=postgres-staging) \
  psql -U appuser appdb_staging < /tmp/prod-dump.sql

echo "Staging database refreshed from production"
```

## Automated Staging Deployment

Configure CI to deploy to staging on every merge to `develop`:

```yaml
# GitHub Actions example
deploy-staging:
  runs-on: ubuntu-latest
  if: github.ref == 'refs/heads/develop'
  steps:
    - name: Trigger Portainer staging deploy
      run: |
        curl -fsS -X POST "${{ secrets.PORTAINER_STAGING_WEBHOOK }}"
```

## Testing in Staging

Run automated tests against the staging URL after deployment:

```bash
# Smoke tests
curl -fsS https://staging.example.com/health | jq .status

# Run full test suite against staging
CYPRESS_BASE_URL=https://staging.example.com npx cypress run

# Load test
k6 run --env BASE_URL=https://staging.example.com load-test.js
```

## Promoting to Production

After tests pass, promote by redeploying production with the same image tag used in staging:

```bash
# Tag the validated staging image as production-ready
docker tag myregistry.example.com/my-app:staging \
           myregistry.example.com/my-app:production

docker push myregistry.example.com/my-app:production

# Trigger production deployment
curl -X POST "$PORTAINER_PROD_WEBHOOK"
```

## Cleanup of Old Staging Deployments

Regularly prune unused staging images to reclaim disk space:

```bash
# Remove dangling images
docker image prune -f

# Remove staging images older than 7 days
docker images --format "{{.Repository}}:{{.Tag}} {{.CreatedAt}}" | \
  grep staging | awk '{print $1}' | xargs docker rmi 2>/dev/null || true
```
