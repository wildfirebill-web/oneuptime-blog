# How to Use OpenID Connect Bearer Middleware with Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, OpenID Connect, Authentication, Middleware, Security

Description: Learn how to configure the Dapr OpenID Connect bearer middleware to validate JWT tokens and authenticate requests to your microservices without custom auth code.

---

## What Is the Dapr OIDC Bearer Middleware

The Dapr OpenID Connect (OIDC) bearer middleware is a Dapr HTTP middleware component that validates JWT bearer tokens on incoming requests to your service. When enabled, Dapr intercepts requests, validates the token against your OIDC provider (Auth0, Keycloak, Azure AD, etc.), and rejects unauthorized requests before they reach your application.

This offloads JWT validation from your application to the Dapr sidecar.

## Prerequisites

- Dapr installed on Kubernetes (or self-hosted)
- An OIDC-compliant identity provider (Auth0, Keycloak, Azure AD, Google, etc.)
- Basic familiarity with JWTs and OIDC

## Define the OIDC Middleware Component

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: oidc-auth
  namespace: default
spec:
  type: middleware.http.bearer
  version: v1
  metadata:
  - name: issuerURL
    value: "https://my-tenant.auth0.com/"
  - name: audience
    value: "https://api.my-app.com"
  - name: jwksURL
    value: "https://my-tenant.auth0.com/.well-known/jwks.json"
```

For Keycloak:

```yaml
  metadata:
  - name: issuerURL
    value: "https://keycloak.example.com/realms/my-realm"
  - name: audience
    value: "my-api-client"
  - name: jwksURL
    value: "https://keycloak.example.com/realms/my-realm/protocol/openid-connect/certs"
```

For Azure AD:

```yaml
  metadata:
  - name: issuerURL
    value: "https://login.microsoftonline.com/YOUR_TENANT_ID/v2.0"
  - name: audience
    value: "api://your-app-id"
  - name: jwksURL
    value: "https://login.microsoftonline.com/YOUR_TENANT_ID/discovery/v2.0/keys"
```

## Create a Middleware Pipeline Configuration

Create a Dapr `Configuration` that applies the middleware to your service:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: my-service-config
  namespace: default
spec:
  httpPipeline:
    handlers:
    - name: oidc-auth
      type: middleware.http.bearer
```

## Apply the Configuration to Your Service

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-api-service
spec:
  template:
    metadata:
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "my-api-service"
        dapr.io/app-port: "3000"
        dapr.io/config: "my-service-config"
```

## Test Authentication

Send a request without a token (should be rejected):

```bash
curl http://my-api-service-url/api/data
# Expected: 401 Unauthorized
```

Send a request with a valid JWT:

```bash
TOKEN="eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9..."

curl -H "Authorization: Bearer $TOKEN" \
  http://my-api-service-url/api/data
# Expected: 200 OK
```

## Access Token Claims in Your Application

After Dapr validates the token, it passes the decoded claims to your application in request headers. Read them in Node.js:

```javascript
const express = require('express');
const app = express();

// Dapr forwards validated token claims as headers
app.get('/api/profile', (req, res) => {
  // Standard headers set by Dapr OIDC middleware
  const sub = req.headers['x-forwarded-user'];         // user subject
  const email = req.headers['x-forwarded-email'];      // email claim
  const scopes = req.headers['x-forwarded-scopes'];    // space-separated scopes

  console.log(`Authenticated user: ${sub} (${email})`);

  res.json({
    userId: sub,
    email: email,
    message: 'Profile data for authenticated user'
  });
});

// Scope-based authorization (after Dapr validates the JWT)
app.delete('/api/resources/:id', (req, res) => {
  const scopes = (req.headers['x-forwarded-scopes'] || '').split(' ');

  if (!scopes.includes('write:resources')) {
    return res.status(403).json({ error: 'Insufficient scope' });
  }

  // Proceed with deletion
  res.json({ deleted: true });
});
```

## Access Claims in Python

```python
from fastapi import FastAPI, Request, HTTPException

app = FastAPI()

@app.get("/api/profile")
async def get_profile(request: Request):
    user_id = request.headers.get("x-forwarded-user")
    email = request.headers.get("x-forwarded-email")
    scopes = request.headers.get("x-forwarded-scopes", "").split()

    if not user_id:
        raise HTTPException(status_code=401, detail="Not authenticated")

    return {
        "userId": user_id,
        "email": email,
        "scopes": scopes
    }

@app.post("/api/admin-action")
async def admin_action(request: Request):
    scopes = request.headers.get("x-forwarded-scopes", "").split()

    if "admin" not in scopes:
        raise HTTPException(status_code=403, detail="Admin scope required")

    return {"result": "Action performed"}
```

## Configure Public Endpoints

Some endpoints (like health checks) should not require authentication. Create a separate configuration without the middleware:

```yaml
# public-config.yaml - no auth middleware
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: public-config
  namespace: default
spec:
  httpPipeline:
    handlers: []   # no middleware
```

Or handle it at the application level by checking if the user header is present.

## Summary

The Dapr OIDC bearer middleware provides automatic JWT validation for incoming requests without modifying application code. By defining the middleware component with your OIDC provider's issuer URL and JWKS endpoint, and referencing it in a Configuration pipeline, Dapr rejects unauthenticated requests at the sidecar level and forwards validated token claims to your service as HTTP headers for authorization decisions.
