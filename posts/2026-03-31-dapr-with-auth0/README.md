# How to Use Dapr with Auth0

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Auth0, Authentication, JWT, OAuth2

Description: Integrate Auth0 with Dapr middleware to validate tokens and forward user identity to microservices without writing auth code in each service.

---

## Auth0 and Dapr Integration

Auth0 issues JWTs that your Dapr sidecar can validate using the bearer token middleware. Once validated, the sidecar forwards claims as request headers, so individual services never handle raw tokens.

## Auth0 Application Setup

```bash
# In Auth0 Dashboard:
# 1. Create a new API (Applications > APIs > Create API)
#    - Name: My Microservices API
#    - Identifier: https://api.myapp.com
#    - Algorithm: RS256

# 2. Note your Auth0 domain: your-tenant.auth0.com
# 3. The JWKS endpoint: https://your-tenant.auth0.com/.well-known/jwks.json
```

## Dapr JWT Middleware for Auth0

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: auth0-validator
  namespace: default
spec:
  type: middleware.http.bearer
  version: v1
  metadata:
  - name: jwksURL
    value: "https://your-tenant.auth0.com/.well-known/jwks.json"
  - name: audience
    value: "https://api.myapp.com"
  - name: issuer
    value: "https://your-tenant.auth0.com/"
```

## Applying to the HTTP Pipeline

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: auth0-config
  namespace: default
spec:
  httpPipeline:
    handlers:
    - name: auth0-validator
      type: middleware.http.bearer
```

## Deployment Annotation

```yaml
metadata:
  annotations:
    dapr.io/enabled: "true"
    dapr.io/app-id: "api-service"
    dapr.io/app-port: "8080"
    dapr.io/config: "auth0-config"
```

## Getting a Token from Auth0

```bash
# Machine-to-machine (M2M) token for service accounts
curl -X POST https://your-tenant.auth0.com/oauth/token \
  -H "Content-Type: application/json" \
  -d '{
    "client_id": "YOUR_CLIENT_ID",
    "client_secret": "YOUR_CLIENT_SECRET",
    "audience": "https://api.myapp.com",
    "grant_type": "client_credentials"
  }'
```

## Reading Auth0 Claims in Your Service

```javascript
const express = require("express");
const app = express();

app.get("/api/me", (req, res) => {
  // Dapr forwards validated Auth0 token claims as headers
  const userId = req.headers["x-jwt-sub"];
  const email = req.headers["x-jwt-https://api.myapp.com/email"];
  const permissions = req.headers["x-jwt-permissions"]?.split(" ") ?? [];

  res.json({ userId, email, permissions });
});

app.get("/api/admin", (req, res) => {
  const permissions = req.headers["x-jwt-permissions"]?.split(" ") ?? [];
  if (!permissions.includes("admin:read")) {
    return res.status(403).json({ error: "insufficient permissions" });
  }
  res.json({ data: "admin data" });
});

app.listen(8080);
```

## Adding Custom Claims via Auth0 Actions

```javascript
// Auth0 Action: add custom claims to tokens
exports.onExecutePostLogin = async (event, api) => {
  const namespace = "https://api.myapp.com";
  api.idToken.setCustomClaim(`${namespace}/email`, event.user.email);
  api.accessToken.setCustomClaim(`${namespace}/roles`, event.authorization?.roles ?? []);
};
```

## Summary

Auth0 and Dapr integrate cleanly: Auth0 issues signed JWTs, Dapr validates them using the JWKS endpoint, and validated claims arrive at your service as HTTP headers. This keeps auth concerns completely outside your application code and makes it trivial to swap identity providers by updating the Dapr middleware component.
