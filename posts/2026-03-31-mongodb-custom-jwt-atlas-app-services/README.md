# How to Implement Custom JWT Authentication in Atlas App Services

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas, JWT, Authentication, Security

Description: Learn how to implement custom JWT authentication in MongoDB Atlas App Services to integrate your own identity provider with Atlas backend services.

---

## Overview

Atlas App Services supports custom JWT authentication, allowing you to integrate any external identity provider that issues JSON Web Tokens - including Auth0, Okta, Firebase Auth, or your own service. Atlas verifies the JWT signature and uses claims to identify users.

## When to Use Custom JWT

Use custom JWT when:
- You have an existing identity provider and don't want to duplicate user management
- You need claims from your own auth system (roles, organization IDs, etc.)
- You want seamless SSO with an existing application

## Configuring Custom JWT in Atlas UI

1. Go to **App Services** and open your application
2. Navigate to **Authentication** and click **Add Authentication Provider**
3. Select **Custom JWT Authentication**
4. Configure the signing algorithm and verification key

## Configuring with a JWKS URI

If your identity provider publishes a JWKS endpoint, use URI verification.

```json
{
  "name": "custom-jwt",
  "type": "custom-token",
  "config": {
    "audience": ["my-atlas-app"],
    "jwksUri": "https://your-idp.example.com/.well-known/jwks.json",
    "signingAlgorithm": "RS256"
  }
}
```

## Configuring with a Signing Secret

For symmetric keys, provide the secret directly.

```json
{
  "name": "custom-jwt",
  "type": "custom-token",
  "config": {
    "signingAlgorithm": "HS256",
    "signingKey": "your-shared-secret-key",
    "audience": ["my-atlas-app"]
  }
}
```

## Mapping JWT Claims to User Metadata

Define metadata fields from JWT claims to populate the Atlas user profile.

```json
{
  "metadataFields": [
    {
      "required": true,
      "name": "email",
      "fieldName": "email"
    },
    {
      "required": false,
      "name": "org_id",
      "fieldName": "org_id"
    }
  ]
}
```

## Generating a JWT for Testing

```javascript
const jwt = require("jsonwebtoken");

const token = jwt.sign(
  {
    sub: "user-123",
    email: "alice@example.com",
    org_id: "org-456",
    aud: "my-atlas-app"
  },
  "your-shared-secret-key",
  {
    algorithm: "HS256",
    expiresIn: "1h"
  }
);

console.log(token);
```

## Authenticating with the Realm SDK

```javascript
import * as Realm from "realm-web";

const app = new Realm.App({ id: "my-app-id" });

const credentials = Realm.Credentials.jwt(jwtToken);
const user = await app.logIn(credentials);

console.log("Logged in as:", user.id);
```

## Using Claims in Data Access Rules

Reference custom claims in collection rules.

```json
{
  "roles": [
    {
      "name": "org-member",
      "apply_when": {
        "org_id": "%%user.custom_data.org_id"
      },
      "read": true,
      "write": false
    }
  ]
}
```

## Summary

Custom JWT authentication in Atlas App Services lets you integrate any JWT-issuing identity provider. Configure verification via JWKS URI or shared secret, map JWT claims to user metadata, and reference those metadata fields in access rules for fine-grained, claim-based data authorization.
