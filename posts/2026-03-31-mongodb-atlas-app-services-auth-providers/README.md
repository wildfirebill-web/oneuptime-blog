# How to Configure Atlas App Services Authentication Providers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas, Authentication, App Services, Security

Description: Learn how to configure authentication providers in MongoDB Atlas App Services including email/password, API key, Google, and custom JWT to secure your application.

---

Atlas App Services supports multiple authentication providers that you can enable and configure independently. Each provider issues an Atlas user session that grants access to App Services resources based on your data access rules.

## Available Authentication Providers

- Email/Password - built-in username and password
- API Key - server-to-server authentication
- Anonymous - temporary sessions without credentials
- Google, Facebook, Apple - OAuth 2.0 social login
- Custom JWT - integrate any external identity provider
- Custom Function - write your own authentication logic

## Enable Email/Password Authentication

In the Atlas UI, navigate to **App Services > Authentication > Email/Password** and enable it. Configure:

```json
{
  "name": "local-userpass",
  "type": "local-userpass",
  "config": {
    "emailConfirmationUrl": "https://yourapp.com/confirm",
    "resetPasswordUrl": "https://yourapp.com/reset-password",
    "confirmEmailSubject": "Confirm Your Email",
    "resetPasswordSubject": "Reset Your Password",
    "runConfirmationFunction": false,
    "runResetFunction": false,
    "autoConfirm": false
  }
}
```

Use the Realm SDK to register and log in:

```javascript
const app = new Realm.App({ id: "<App ID>" })

// Register a new user
await app.emailPasswordAuth.registerUser({ email, password })

// Log in
const credentials = Realm.Credentials.emailPassword(email, password)
const user = await app.logIn(credentials)
```

## Enable API Key Authentication

API key auth is ideal for server-side or CI/CD processes:

```javascript
// Create an API key for a logged-in user
const apiKey = await user.apiKeys.create("my-server-key")
console.log(apiKey.key) // Save this - it won't be shown again
```

Authenticate using the key:

```javascript
const credentials = Realm.Credentials.apiKey(apiKey.key)
const user = await app.logIn(credentials)
```

## Enable Google OAuth

In the Atlas UI, enable the Google provider and configure your OAuth 2.0 Client ID and Client Secret from the Google Cloud Console:

```json
{
  "name": "oauth2-google",
  "type": "oauth2-google",
  "config": {
    "clientId": "<GOOGLE_CLIENT_ID>"
  },
  "secret_config": {
    "clientSecret": "googleOAuthClientSecret"
  }
}
```

In your web app:

```javascript
const credentials = Realm.Credentials.google({ idToken: googleIdToken })
const user = await app.logIn(credentials)
```

## Custom JWT Provider

Connect any external identity provider (Auth0, Cognito, Okta) by configuring Custom JWT:

```json
{
  "name": "custom-token",
  "type": "custom-token",
  "config": {
    "audience": "https://yourapp.com",
    "signingAlgorithm": "RS256",
    "useJWKURI": true,
    "jwkURI": "https://your-idp.com/.well-known/jwks.json",
    "metadataFields": [
      { "required": true, "name": "email", "field_name": "email" }
    ]
  }
}
```

Authenticate with the JWT from your identity provider:

```javascript
const credentials = Realm.Credentials.jwt(externalJwtToken)
const user = await app.logIn(credentials)
```

## Link Multiple Providers

Users can link multiple authentication identities to one Atlas account:

```javascript
const googleCredentials = Realm.Credentials.google({ idToken })
await user.linkCredentials(googleCredentials)
```

## Summary

Atlas App Services supports a wide range of authentication providers configurable through the UI or JSON config files. Enable email/password for direct user management, API keys for server processes, OAuth providers for social login, and custom JWT to integrate existing identity providers. Store provider configuration in version control and deploy with the App Services CLI for consistent environments.
