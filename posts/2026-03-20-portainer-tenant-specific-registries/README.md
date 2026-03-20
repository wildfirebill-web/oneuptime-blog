# How to Set Up Tenant-Specific Registries in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Registry, Multi-Tenancy, Teams, Docker Hub, Private Registry

Description: Learn how to configure tenant-specific Docker registries in Portainer so each team can only access their own images and registry credentials.

---

In multi-tenant Portainer deployments, different teams often have their own private registries or separate organizational accounts. Portainer allows you to add multiple registries and restrict access to specific teams so tenants cannot use each other's registries or credentials.

## Adding a Registry in Portainer

Go to **Registries > Add registry** and select the registry type:

- **DockerHub** - requires username and access token
- **AWS ECR** - uses IAM credentials
- **GitLab Container Registry** - uses deploy token
- **Custom** - any Docker-compatible registry

## Configuring Registry Access per Team

After adding a registry, restrict which teams can use it:

1. In Portainer, go to **Registries > [registry name]**.
2. Click **Manage access**.
3. Under **Teams access**, add only the teams that should have access.
4. Teams not listed will not see or be able to use this registry.

## Registry Setup Example: Two Tenants

Configure two separate registries for two tenant teams:

```bash
TOKEN="your-admin-jwt-token"
PORTAINER="https://portainer.example.com"

# Add Tenant A's registry

REG_A_ID=$(curl -s -X POST "$PORTAINER/api/registries" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "Name": "Tenant A Registry",
    "Type": 1,
    "URL": "registry-a.example.com",
    "Authentication": true,
    "Username": "tenant-a-user",
    "Password": "tenant-a-token"
  }' | jq -r .Id)

# Restrict to Team A only
curl -s -X PUT "$PORTAINER/api/registries/$REG_A_ID/configure" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"TeamAccessPolicies\": {\"$TEAM_A_ID\": {\"RoleId\": 1}}}"
```

## AWS ECR Per-Tenant Configuration

For teams using separate AWS accounts with ECR, add one registry entry per account:

```bash
# Tenant A uses AWS account 111122223333
curl -s -X POST "$PORTAINER/api/registries" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "Name": "Tenant A ECR",
    "Type": 7,
    "URL": "111122223333.dkr.ecr.us-east-1.amazonaws.com",
    "Ecr": {
      "Region": "us-east-1"
    },
    "Authentication": true,
    "Username": "AWS",
    "Password": "<IAM-access-key-id>",
    "AWSAccessKeyID": "<key-id>",
    "AWSSecretAccessKey": "<secret-key>",
    "AWSRegion": "us-east-1"
  }'
```

ECR tokens expire every 12 hours - Portainer Business Edition auto-refreshes them.

## Self-Hosted Registry per Tenant

Deploy a separate registry container for each tenant for full isolation:

```yaml
# Tenant A registry stack
version: "3.8"

services:
  registry-a:
    image: registry:2
    environment:
      REGISTRY_HTTP_SECRET: tenant-a-secret
      REGISTRY_AUTH: htpasswd
      REGISTRY_AUTH_HTPASSWD_REALM: "Tenant A Registry"
      REGISTRY_AUTH_HTPASSWD_PATH: /auth/htpasswd
    volumes:
      - registry_a_data:/var/lib/registry
      - ./auth:/auth
    ports:
      - "5001:5000"   # Different port per tenant

volumes:
  registry_a_data:
```

Generate htpasswd credentials:

```bash
docker run --rm httpd:2 htpasswd -nb tenant-a-user "securepassword" > auth/htpasswd
```

## Verifying Registry Isolation

Log in as a Tenant A user and verify they can only see and use Tenant A's registry:

```bash
TENANT_A_TOKEN=$(curl -s -X POST "$PORTAINER/api/auth" \
  -d '{"Username":"alice","Password":"pass"}' \
  -H 'Content-Type: application/json' | jq -r .jwt)

# This should only return Tenant A's registry
curl -s -H "Authorization: Bearer $TENANT_A_TOKEN" \
  "$PORTAINER/api/registries" | jq '.[].Name'
```

Tenant A users cannot see or use Tenant B's registry credentials, even if they know the URL.
