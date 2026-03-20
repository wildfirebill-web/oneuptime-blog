# How to Deploy Keycloak via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Keycloak, SSO, Identity Provider, Self-Hosted, OAuth

Description: Deploy Keycloak via Portainer as a comprehensive single sign-on and identity management solution supporting OAuth 2.0, OpenID Connect, and SAML.

## Introduction

Keycloak is an open-source Identity and Access Management solution providing SSO, social login, and identity brokering. It supports OAuth 2.0, OpenID Connect, and SAML 2.0 protocols. Deploying via Portainer with PostgreSQL gives you a production-ready identity provider.

## Deploy as a Stack

```yaml
version: "3.8"

services:
  keycloak:
    image: quay.io/keycloak/keycloak:24.0
    container_name: keycloak
    command: start --optimized
    environment:
      # Admin credentials
      KC_BOOTSTRAP_ADMIN_USERNAME: admin
      KC_BOOTSTRAP_ADMIN_PASSWORD: admin_password
      
      # Database
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://keycloak-db:5432/keycloak
      KC_DB_USERNAME: keycloak
      KC_DB_PASSWORD: keycloak_db_password
      
      # Hostname (change to your domain)
      KC_HOSTNAME: keycloak.example.com
      KC_HOSTNAME_STRICT: "false"
      KC_HTTP_ENABLED: "true"
      
      # Proxy settings (if behind reverse proxy)
      KC_PROXY: edge
      
      # Performance
      KC_HEALTH_ENABLED: "true"
      KC_METRICS_ENABLED: "true"
    ports:
      - "8080:8080"
      - "8443:8443"
    depends_on:
      keycloak-db:
        condition: service_healthy
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "curl -s http://localhost:8080/health/ready | grep -q 'UP'"]
      interval: 30s
      timeout: 10s
      retries: 10
      start_period: 120s

  keycloak-db:
    image: postgres:16-alpine
    container_name: keycloak-db
    environment:
      POSTGRES_DB: keycloak
      POSTGRES_USER: keycloak
      POSTGRES_PASSWORD: keycloak_db_password
    volumes:
      - keycloak_db:/var/lib/postgresql/data
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U keycloak"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  keycloak_db:
```

## Initial Configuration

1. Access Keycloak at `http://<host>:8080`
2. Click **Administration Console**
3. Log in with `admin/admin_password`

## Creating a Realm

1. Click **Create realm** in the top-left dropdown
2. Name: `mycompany`
3. Enable the realm

## Creating an OAuth2 Client

1. Navigate to **Clients > Create client**
2. Client type: **OpenID Connect**
3. Client ID: `myapp`
4. Authentication flow: **Standard flow** (for web apps)
5. Set **Valid redirect URIs**: `https://app.example.com/callback`
6. Set **Web origins**: `https://app.example.com`

## Creating Users

```bash
# Via Keycloak Admin API
TOKEN=$(curl -s http://keycloak:8080/realms/master/protocol/openid-connect/token \
  -d "client_id=admin-cli&username=admin&password=admin_password&grant_type=password" \
  | jq -r '.access_token')

# Create user
curl -X POST "http://keycloak:8080/admin/realms/mycompany/users" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "john.doe",
    "email": "john@example.com",
    "enabled": true,
    "credentials": [{"type": "password", "value": "user_password", "temporary": false}]
  }'
```

## Integrating Applications

### OAuth2 with Python (Flask)

```python
from flask import Flask, redirect, url_for
from authlib.integrations.flask_client import OAuth

app = Flask(__name__)
oauth = OAuth(app)

keycloak = oauth.register(
    name='keycloak',
    client_id='myapp',
    client_secret='your-client-secret',
    server_metadata_url='http://keycloak:8080/realms/mycompany/.well-known/openid-configuration',
    client_kwargs={'scope': 'openid email profile'}
)

@app.route('/login')
def login():
    return keycloak.authorize_redirect(url_for('callback', _external=True))

@app.route('/callback')
def callback():
    token = keycloak.authorize_access_token()
    user = token['userinfo']
    return f"Logged in as: {user['email']}"
```

## Conclusion

Keycloak deployed via Portainer provides enterprise-grade SSO and identity management for your self-hosted application stack. Once configured, you can add social login, federate with LDAP/Active Directory, and provide a single authentication point for all your applications. The Admin API makes user management scriptable for bulk operations.
