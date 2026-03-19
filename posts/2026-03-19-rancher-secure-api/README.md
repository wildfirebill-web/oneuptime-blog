# How to Secure Rancher API Access

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Security

Description: Learn how to secure the Rancher API with authentication, authorization, rate limiting, and access controls to prevent unauthorized access.

The Rancher API provides programmatic access to your entire Kubernetes infrastructure. A compromised API key can give an attacker control over all your clusters. Securing API access is critical for protecting your Rancher deployment. This guide covers hardening Rancher API access with multiple layers of protection.

## Prerequisites

- Rancher v2.5 or later
- Admin access to Rancher
- kubectl access with cluster admin privileges
- Understanding of Rancher RBAC model

## Step 1: Use Scoped API Keys Instead of Admin Tokens

Never use the admin token for automation. Create scoped API keys with minimum required permissions.

### Create a Scoped API Key via UI

1. Log in to Rancher as the appropriate user (not admin).
2. Click the user icon > **API & Keys**.
3. Click **Create API Key**.
4. Set a **Description** for the key.
5. Set an **Expiration** (e.g., 90 days).
6. Under **Scope**, select the specific cluster or project.
7. Click **Create**.

### Create a Scoped API Key via CLI

```bash
curl -X POST \
  'https://rancher.yourdomain.com/v3/tokens' \
  -H 'Authorization: Bearer ADMIN_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
    "type": "token",
    "description": "CI/CD pipeline - cluster-1 deploy only",
    "ttl": 7776000000,
    "userId": "user-xxxxx"
  }'
```

## Step 2: Set Token Expiration Policies

Configure maximum token lifetime to prevent long-lived credentials:

```bash
helm upgrade rancher rancher-latest/rancher \
  -n cattle-system \
  --set 'extraEnv[0].name=CATTLE_TOKEN_MAX_TTL_MINUTES' \
  --set 'extraEnv[0].value=43200'
```

This sets a maximum token lifetime of 30 days (43200 minutes).

Through the Rancher UI:

1. Go to **Global Settings**.
2. Find **auth-token-max-ttl-minutes**.
3. Set to your desired maximum (e.g., `43200` for 30 days).

## Step 3: Restrict API Access by IP

Use network-level controls to limit which IP addresses can reach the Rancher API.

### Using an Ingress Controller

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rancher
  namespace: cattle-system
  annotations:
    nginx.ingress.kubernetes.io/whitelist-source-range: "10.0.0.0/8,172.16.0.0/12,203.0.113.0/24"
spec:
  rules:
  - host: rancher.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: rancher
            port:
              number: 443
```

### Using a Network Policy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-rancher-api
  namespace: cattle-system
spec:
  podSelector:
    matchLabels:
      app: rancher
  policyTypes:
  - Ingress
  ingress:
  - from:
    - ipBlock:
        cidr: 10.0.0.0/8
    - ipBlock:
        cidr: 203.0.113.0/24
    ports:
    - protocol: TCP
      port: 443
```

## Step 4: Enable Rate Limiting

Protect the API from brute force attacks and abuse with rate limiting.

### Using NGINX Ingress Controller

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rancher
  namespace: cattle-system
  annotations:
    nginx.ingress.kubernetes.io/limit-rps: "10"
    nginx.ingress.kubernetes.io/limit-burst-multiplier: "5"
    nginx.ingress.kubernetes.io/limit-connections: "5"
spec:
  rules:
  - host: rancher.yourdomain.com
    http:
      paths:
      - path: /v3/tokens
        pathType: Prefix
        backend:
          service:
            name: rancher
            port:
              number: 443
```

This limits authentication endpoints to 10 requests per second per client.

## Step 5: Configure External Authentication

Replace local authentication with an external identity provider for stronger security:

### LDAP/Active Directory

1. Go to **Users & Authentication** > **Auth Provider**.
2. Select **ActiveDirectory**.
3. Configure the LDAP connection:

```plaintext
Server: ldap.yourdomain.com
Port: 636
TLS: true
Service Account DN: cn=rancher,ou=service,dc=yourdomain,dc=com
Search Base: dc=yourdomain,dc=com
```

4. Test and enable.

### SAML (Okta, Azure AD, etc.)

1. Go to **Users & Authentication** > **Auth Provider**.
2. Select **SAML**.
3. Configure with your identity provider's metadata URL.
4. Map SAML attributes to Rancher roles.

### After enabling external auth, disable local authentication:

```bash
kubectl -n cattle-system patch setting disable-local-auth \
  --type='merge' -p '{"value":"true"}'
```

## Step 6: Implement Service Account Best Practices

For automation and CI/CD, use dedicated service accounts:

```bash
# Create a service account in Rancher for CI/CD
curl -X POST \
  'https://rancher.yourdomain.com/v3/users' \
  -H 'Authorization: Bearer ADMIN_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
    "type": "user",
    "username": "cicd-deployer",
    "mustChangePassword": false,
    "enabled": true
  }'
```

Assign minimal permissions:

```bash
# Grant deploy-only access to a specific project
curl -X POST \
  'https://rancher.yourdomain.com/v3/projectroletemplatebindings' \
  -H 'Authorization: Bearer ADMIN_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
    "type": "projectRoleTemplateBinding",
    "projectId": "c-xxxxx:p-xxxxx",
    "roleTemplateId": "project-member",
    "userId": "user-xxxxx"
  }'
```

## Step 7: Audit API Access

Enable and monitor API access logs. Ensure audit logging is configured (see the audit logging guide) and specifically monitor:

```bash
# Check recent API token usage
kubectl logs -n cattle-system -l app=rancher -c rancher-audit-log | \
  jq 'select(.requestURI | startswith("/v3"))' | head -20
```

Monitor for suspicious patterns:

- Multiple failed authentication attempts from the same IP
- API access from unexpected geographic locations
- High-volume API calls from a single token
- Access to sensitive endpoints (tokens, secrets, cluster registrations)

## Step 8: Secure the Kubernetes API Endpoint

In addition to the Rancher API, secure the underlying Kubernetes API:

### Disable Anonymous Authentication

```yaml
# RKE2 config
kube-apiserver-arg:
  - "anonymous-auth=false"
```

### Enable OIDC Authentication

```yaml
kube-apiserver-arg:
  - "oidc-issuer-url=https://auth.yourdomain.com"
  - "oidc-client-id=kubernetes"
  - "oidc-username-claim=email"
  - "oidc-groups-claim=groups"
```

### Restrict API Server Network Access

On cloud providers, ensure the API server load balancer is not publicly accessible:

```bash
# AWS - restrict security group
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxxxx \
  --protocol tcp \
  --port 6443 \
  --cidr 10.0.0.0/8
```

## Step 9: Rotate API Tokens Regularly

Implement a token rotation process:

```bash
#!/bin/bash
# Script to rotate API tokens

# List tokens older than 60 days
TOKENS=$(curl -s \
  'https://rancher.yourdomain.com/v3/tokens' \
  -H 'Authorization: Bearer ADMIN_TOKEN' | \
  jq -r '.data[] | select(.expired == false) | select((.createdTS / 1000) < (now - 5184000)) | .id')

for TOKEN_ID in $TOKENS; do
  echo "Revoking old token: $TOKEN_ID"
  curl -X DELETE \
    "https://rancher.yourdomain.com/v3/tokens/$TOKEN_ID" \
    -H 'Authorization: Bearer ADMIN_TOKEN'
done
```

Schedule this as a periodic job.

## Step 10: Restrict Default Admin Access

After setting up external authentication and creating appropriate admin accounts, restrict the default admin:

1. Change the default admin password to a long, random string.
2. Store it in a secure vault for emergency access only.
3. Enable the restrict default admin setting:

```bash
kubectl -n cattle-system patch setting auth-user-info-max-age-seconds \
  --type='merge' -p '{"value":"0"}'
```

## Security Checklist

- [ ] Use scoped API keys, not admin tokens
- [ ] Set token expiration policies
- [ ] Restrict API access by IP
- [ ] Enable rate limiting
- [ ] Configure external authentication (LDAP/SAML)
- [ ] Disable local authentication
- [ ] Audit all API access
- [ ] Rotate tokens regularly
- [ ] Disable anonymous Kubernetes API access
- [ ] Restrict default admin account

## Conclusion

Securing the Rancher API requires a defense-in-depth approach combining authentication, authorization, network controls, and monitoring. By using scoped tokens, external authentication, IP restrictions, and rate limiting, you significantly reduce the risk of unauthorized access to your Kubernetes infrastructure. Regular token rotation and audit logging provide ongoing protection and compliance visibility.
