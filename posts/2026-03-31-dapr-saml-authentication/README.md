# How to Use Dapr with SAML Authentication

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, SAML, Authentication, Security, Middleware

Description: Integrate SAML authentication with Dapr-enabled services by using a SAML-to-OIDC bridge or reverse proxy to validate identity assertions from enterprise identity providers.

---

## SAML and Dapr Integration

Dapr does not have native SAML support, but you can integrate SAML authentication by placing a SAML service provider (SP) in front of your Dapr-enabled services. The SP validates SAML assertions from your Identity Provider (IdP) and converts them to JWT tokens that Dapr's OAuth2 middleware can validate.

## Architecture Pattern

The recommended architecture is:

```bash
Client -> SAML SP (e.g., oauth2-proxy or Authelia) -> Dapr Sidecar -> App
```

The SAML SP handles the IdP redirect flow, validates the SAML assertion, and sets a session cookie or Authorization header that downstream Dapr middleware validates.

## Setting Up Authelia as SAML SP

Authelia supports SAML 2.0 and can issue OAuth2/OIDC tokens after SAML authentication:

```yaml
# authelia-config.yaml
server:
  host: 0.0.0.0
  port: 9091

authentication_backend:
  ldap:
    url: ldap://ldap.example.com
    base_dn: dc=example,dc=com

session:
  domain: example.com

identity_providers:
  oidc:
    hmac_secret: very-secret-key
    issuer_private_key: /secrets/oidc.key
    clients:
      - id: dapr-app
        secret: client-secret
        scopes:
          - openid
          - profile
          - groups
        redirect_uris:
          - https://myapp.example.com/callback
```

## Configuring Dapr to Validate JWT After SAML Auth

After the SAML SP issues a JWT, configure Dapr middleware to validate it:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: jwt-validator
spec:
  type: middleware.http.bearer
  version: v1
  metadata:
    - name: jwksURL
      value: "https://auth.example.com/.well-known/jwks.json"
    - name: audience
      value: "dapr-app"
    - name: issuer
      value: "https://auth.example.com"
```

Apply via pipeline:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: app-config
spec:
  httpPipeline:
    handlers:
      - name: jwt-validator
        type: middleware.http.bearer
```

## Extracting SAML Attributes from JWT

After SAML authentication, the IdP passes attributes (groups, email, role) into the JWT claims:

```python
import jwt
from functools import wraps
from flask import request, jsonify

def require_saml_group(group_name: str):
    def decorator(f):
        @wraps(f)
        def decorated_function(*args, **kwargs):
            token = request.headers.get("Authorization", "").replace("Bearer ", "")
            claims = jwt.decode(token, options={"verify_signature": False})

            groups = claims.get("groups", [])
            if group_name not in groups:
                return jsonify({"error": "Insufficient permissions"}), 403

            return f(*args, **kwargs)
        return decorated_function
    return decorator

@app.route('/admin/report')
@require_saml_group('dapr-admins')
def admin_report():
    return jsonify({"report": "data"})
```

## Testing SAML Authentication Flow

```bash
# Initiate SAML login flow
curl -v https://myapp.example.com/api/data

# Should redirect to IdP login page
# After login, IdP posts SAML assertion to SP
# SP issues JWT and redirects to original URL
```

## Summary

Integrate SAML authentication with Dapr by using a SAML service provider like Authelia as a reverse proxy that converts SAML assertions into JWT tokens. Configure Dapr's bearer token middleware to validate the JWT and extract user claims. Application code reads SAML attributes (groups, roles) from the validated JWT claims for authorization decisions.
