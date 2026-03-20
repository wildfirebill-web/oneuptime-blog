# How to Deploy Authentik via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Authentik, SSO, Identity Provider, OAuth, Self-Hosted

Description: Deploy Authentik via Portainer as a modern, feature-rich identity provider supporting OAuth2, OIDC, SAML, and LDAP with a polished admin interface.

## Introduction

Authentik is a modern Identity Provider that's easier to configure than Keycloak while providing more features than Authelia. It supports OAuth2, OpenID Connect, SAML, LDAP, SCIM, and includes a polished web UI. Deploy via Portainer with PostgreSQL and Redis for a complete IAM solution.

## Deploy as a Stack

```yaml
version: "3.8"

services:
  authentik-server:
    image: ghcr.io/goauthentik/server:2024.6
    container_name: authentik-server
    command: server
    environment:
      AUTHENTIK_REDIS__HOST: authentik-redis
      AUTHENTIK_POSTGRESQL__HOST: authentik-db
      AUTHENTIK_POSTGRESQL__USER: authentik
      AUTHENTIK_POSTGRESQL__NAME: authentik
      AUTHENTIK_POSTGRESQL__PASSWORD: db_password
      AUTHENTIK_SECRET_KEY: your_very_secret_key_change_this
      AUTHENTIK_ERROR_REPORTING__ENABLED: "false"
      AUTHENTIK_EMAIL__HOST: smtp.example.com
      AUTHENTIK_EMAIL__PORT: 587
      AUTHENTIK_EMAIL__FROM: authentik@example.com
      AUTHENTIK_EMAIL__USERNAME: authentik@example.com
      AUTHENTIK_EMAIL__PASSWORD: smtp_password
      AUTHENTIK_EMAIL__USE_TLS: "true"
    volumes:
      - authentik_media:/media
      - authentik_templates:/templates
    ports:
      - "9000:9000"    # HTTP
      - "9443:9443"    # HTTPS
    depends_on:
      - authentik-db
      - authentik-redis
    restart: unless-stopped

  authentik-worker:
    image: ghcr.io/goauthentik/server:2024.6
    container_name: authentik-worker
    command: worker
    environment:
      AUTHENTIK_REDIS__HOST: authentik-redis
      AUTHENTIK_POSTGRESQL__HOST: authentik-db
      AUTHENTIK_POSTGRESQL__USER: authentik
      AUTHENTIK_POSTGRESQL__NAME: authentik
      AUTHENTIK_POSTGRESQL__PASSWORD: db_password
      AUTHENTIK_SECRET_KEY: your_very_secret_key_change_this
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - authentik_media:/media
      - authentik_certs:/certs
    depends_on:
      - authentik-db
      - authentik-redis
    restart: unless-stopped

  authentik-db:
    image: postgres:16-alpine
    container_name: authentik-db
    environment:
      POSTGRES_PASSWORD: db_password
      POSTGRES_USER: authentik
      POSTGRES_DB: authentik
    volumes:
      - authentik_db:/var/lib/postgresql/data
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U authentik"]
      interval: 10s
      timeout: 5s
      retries: 5

  authentik-redis:
    image: redis:7-alpine
    container_name: authentik-redis
    command: --save 60 1 --loglevel warning
    volumes:
      - authentik_redis:/data
    restart: unless-stopped

volumes:
  authentik_media:
  authentik_templates:
  authentik_db:
  authentik_redis:
  authentik_certs:
```

## Initial Setup

1. Access `http://<host>:9000/if/flow/initial-setup/`
2. Create the admin user
3. Log in to the admin interface at `http://<host>:9000/`

## Configure OAuth2 Provider

1. Navigate to **Applications > Providers > Create**
2. Select **OAuth2/OpenID Provider**
3. Configure:
   - Name: `MyApp OAuth`
   - Client ID: (auto-generated)
   - Client Secret: (auto-generated)
   - Redirect URIs: `https://app.example.com/auth/callback`
4. Create an **Application** and link the provider

## Configure Proxy Provider (Reverse Proxy Auth)

For services behind a reverse proxy without OAuth support:

1. Create a **Proxy Provider** with Forward Auth mode
2. Set external host: `https://app.example.com`
3. Create an Application and link the provider
4. Add the Authentik outpost for the proxy

## LDAP Provider

Enable LDAP for integrating with other services:

1. Create an **LDAP Provider**
2. Note the LDAP bind DN
3. Configure services to use Authentik as LDAP server

## Social Login Integration

1. Navigate to **Directory > Federation & Social Login**
2. Click **Create GitHub OAuth Source** (or other providers)
3. Register an OAuth app on GitHub and enter client credentials
4. Users can now log in with GitHub accounts

## Generate Secret Key

```bash
# Generate a strong secret key
python3 -c "import secrets; print(secrets.token_hex(50))"
```

## Conclusion

Authentik deployed via Portainer provides a modern, polished identity provider that bridges the gap between lightweight Authelia and full-featured Keycloak. Its excellent UI makes provider and policy configuration intuitive, while the broad protocol support (OAuth2, OIDC, SAML, LDAP) means it integrates with virtually any application. The worker process handles background tasks like email delivery and LDAP sync.
