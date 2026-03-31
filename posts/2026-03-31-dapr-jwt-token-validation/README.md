# How to Implement JWT Token Validation with Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, JWT, Authentication, Security, Middleware

Description: Configure Dapr middleware to validate JWT tokens on incoming requests, extract claims, and enforce authorization without writing custom auth code.

---

## JWT Validation with Dapr Middleware

Dapr's middleware pipeline can validate JWTs before a request reaches your application. This offloads token verification from your services and centralizes the auth logic in the sidecar.

## Configuring the JWT Middleware

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: jwt-validator
  namespace: default
spec:
  type: middleware.http.bearer
  version: v1
  metadata:
  - name: jwksURL
    value: "https://auth.example.com/.well-known/jwks.json"
  - name: audience
    value: "api.example.com"
  - name: issuer
    value: "https://auth.example.com/"
```

## Applying the Middleware via Configuration

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: api-config
  namespace: default
spec:
  httpPipeline:
    handlers:
    - name: jwt-validator
      type: middleware.http.bearer
```

## Attaching the Config to Your Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
spec:
  template:
    metadata:
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "api-service"
        dapr.io/app-port: "8080"
        dapr.io/config: "api-config"
```

## Reading JWT Claims in Your Application

Dapr forwards validated claims as HTTP headers:

```python
from fastapi import FastAPI, Request

app = FastAPI()

@app.get("/api/profile")
async def get_profile(request: Request):
    # Dapr forwards these headers after JWT validation
    user_id = request.headers.get("X-JWT-Sub")
    email = request.headers.get("X-JWT-Email")
    roles = request.headers.get("X-JWT-Roles", "").split(",")

    return {
        "userId": user_id,
        "email": email,
        "roles": roles
    }
```

## Generating a Test JWT

```bash
# Install jwt-cli
npm install -g @clarketm/jwt-cli

# Generate a test token (for development only)
jwt sign \
  --algorithm RS256 \
  --issuer "https://auth.example.com/" \
  --audience "api.example.com" \
  --subject "user-123" \
  --expires "1h" \
  --private-key ./dev-private-key.pem
```

## Testing JWT Validation

```bash
# Valid token - should return 200
TOKEN="eyJhbGci..."
curl -H "Authorization: Bearer $TOKEN" http://localhost:3500/v1.0/invoke/api-service/method/profile

# No token - Dapr returns 401 before request reaches your app
curl http://localhost:3500/v1.0/invoke/api-service/method/profile
# HTTP 401 Unauthorized

# Expired token - Dapr returns 401
curl -H "Authorization: Bearer $EXPIRED_TOKEN" \
  http://localhost:3500/v1.0/invoke/api-service/method/profile
# HTTP 401 Unauthorized
```

## Custom Claim Extraction Middleware

```go
func claimExtractorMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Claims are already validated by Dapr - just extract them
        sub := r.Header.Get("X-JWT-Sub")
        if sub == "" {
            http.Error(w, "missing user context", http.StatusUnauthorized)
            return
        }

        ctx := context.WithValue(r.Context(), "userID", sub)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

## Summary

Dapr's bearer token middleware handles JWT validation including JWKS fetching, token signature verification, expiry checks, and audience/issuer validation. Your application receives only pre-validated requests with claims forwarded as headers, completely decoupling authentication logic from business logic.
