# How to Implement Custom JWT Authentication in Atlas App Services

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB Atlas, JWT, Authentication, App Service, Security

Description: Learn how to implement custom JWT authentication in MongoDB Atlas App Services to integrate your own identity provider and control user access.

---

## Introduction

MongoDB Atlas App Services supports custom JWT authentication, allowing you to use your own identity provider (IdP) or authentication system. When a user authenticates with your backend, you issue a JWT signed with your private key. Atlas verifies the token against your public key or JWKS endpoint and grants access to your Atlas App. This approach keeps your existing auth system while leveraging Atlas for data access.

## How Custom JWT Authentication Works

```text
1. User authenticates with your backend (username/password, OAuth, etc.)
2. Your backend issues a signed JWT with user claims
3. Client sends JWT to Atlas App Services
4. Atlas validates the JWT signature using your public key/JWKS
5. Atlas creates or fetches the matching App Services user
6. User gets an Atlas access token for subsequent requests
```

## Step 1 - Generate JWT Keys

Generate an RS256 key pair for signing JWTs:

```bash
# Generate private key
openssl genrsa -out private.pem 2048

# Generate corresponding public key
openssl rsa -in private.pem -pubout -out public.pem

# View public key in PKCS#8 format (for Atlas)
cat public.pem
```

## Step 2 - Configure Atlas App Services

In Atlas - App Services - App - Authentication - Custom JWT:

```json
{
  "enabled": true,
  "audience": "your-app-audience",
  "signingAlgorithm": "RS256",
  "useJWKSURI": false,
  "signingKey": "<your-RSA-public-key-content>"
}
```

Or use a JWKS endpoint if your IdP provides one:

```json
{
  "useJWKSURI": true,
  "jwksURI": "https://your-auth-server.com/.well-known/jwks.json"
}
```

## Step 3 - Define Metadata Field Mapping

Map JWT claims to Atlas user metadata fields:

```json
{
  "metadataFields": [
    { "required": true, "name": "userId", "field_name": "sub" },
    { "required": false, "name": "email", "field_name": "email" },
    { "required": false, "name": "name", "field_name": "name" },
    { "required": false, "name": "role", "field_name": "role" }
  ]
}
```

## Step 4 - Issue JWTs from Your Backend

Node.js example using `jsonwebtoken`:

```javascript
const jwt = require("jsonwebtoken");
const fs = require("fs");

const privateKey = fs.readFileSync("./private.pem", "utf8");

function generateAtlasJWT(user) {
  const now = Math.floor(Date.now() / 1000);

  const payload = {
    sub: user.id.toString(),      // Subject (required)
    iat: now,                      // Issued at (required)
    exp: now + 3600,               // Expires in 1 hour (required)
    aud: "your-app-audience",      // Must match Atlas config
    email: user.email,
    name: user.name,
    role: user.role
  };

  return jwt.sign(payload, privateKey, { algorithm: "RS256" });
}

// In your auth endpoint
app.post("/auth/token", async (req, res) => {
  const user = await validateCredentials(req.body.email, req.body.password);
  if (!user) return res.status(401).json({ error: "Invalid credentials" });

  const token = generateAtlasJWT(user);
  res.json({ atlasToken: token });
});
```

## Step 5 - Authenticate Client with Atlas

Use the Atlas SDK with the custom JWT:

```javascript
const Realm = require("realm");

async function connectToAtlas(jwtToken) {
  const app = new Realm.App({ id: "your-app-id" });

  const credentials = Realm.Credentials.jwt(jwtToken);
  const user = await app.logIn(credentials);

  console.log("Logged in as:", user.id);
  return user;
}
```

Or with the Web SDK:

```javascript
import * as Realm from "realm-web";

const app = new Realm.App({ id: "your-app-id" });

async function loginWithJWT(jwtToken) {
  const credentials = Realm.Credentials.jwt(jwtToken);
  const user = await app.logIn(credentials);
  return user;
}
```

## Step 6 - Define Data Access Rules Using JWT Claims

Use JWT claims in Atlas Rules to control data access:

```javascript
// Rule for users collection - users can only read their own data
// In Atlas App Services - Rules:
{
  "database": "mydb",
  "collection": "users",
  "roles": [
    {
      "name": "owner",
      "apply_when": { "userId": "%%user.data.userId" },
      "read": true,
      "write": true
    }
  ]
}
```

For admin users with a role claim:

```javascript
{
  "roles": [
    {
      "name": "admin",
      "apply_when": { "%%user.data.role": "admin" },
      "read": true,
      "write": true,
      "insert": true,
      "delete": true
    },
    {
      "name": "user",
      "apply_when": {},
      "read": { "userId": "%%user.data.userId" },
      "write": false
    }
  ]
}
```

## Token Refresh Pattern

JWT tokens expire - handle token refresh in your application:

```javascript
async function getValidToken(user) {
  // Check if Atlas token will expire in next 5 minutes
  const expiresAt = user.accessToken.expiresAt;
  const now = Date.now();

  if (expiresAt - now < 5 * 60 * 1000) {
    // Get fresh JWT from your backend
    const response = await fetch("/auth/refresh", {
      headers: { Authorization: `Bearer ${localStorage.getItem("refreshToken")}` }
    });
    const { atlasToken } = await response.json();

    // Re-login to Atlas with new JWT
    const credentials = Realm.Credentials.jwt(atlasToken);
    await app.currentUser?.logOut();
    await app.logIn(credentials);
  }

  return app.currentUser;
}
```

## Python Client Example

```python
import requests
import json

ATLAS_APP_ID = "your-app-id"

def login_with_jwt(jwt_token):
    url = f"https://realm.mongodb.com/api/client/v2.0/app/{ATLAS_APP_ID}/auth/providers/custom-token/login"
    response = requests.post(url, json={"token": jwt_token})
    response.raise_for_status()
    return response.json()["access_token"]

# Use the access token in GraphQL requests
access_token = login_with_jwt(your_jwt)
headers = {"Authorization": f"Bearer {access_token}"}
```

## Summary

Custom JWT authentication in Atlas App Services bridges your existing identity infrastructure with MongoDB's data access controls. Issue JWTs from your backend with appropriate claims, configure Atlas to validate them against your public key or JWKS endpoint, and map JWT claims to Atlas user metadata for use in access rules. This approach enables fine-grained, claim-based data access control without duplicating user management in Atlas.
