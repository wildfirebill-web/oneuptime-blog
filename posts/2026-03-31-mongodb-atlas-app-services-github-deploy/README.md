# How to Deploy Atlas App Services with GitHub Integration

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas, GitHub, CI/CD, App Service

Description: Learn how to connect MongoDB Atlas App Services to a GitHub repository to enable automatic deployments when you push configuration changes to your main branch.

---

Atlas App Services can automatically deploy configuration changes from a GitHub repository. This keeps your App Services config in version control and enables GitOps workflows where a push to the main branch triggers a deployment.

## Repository Structure

Atlas App Services configuration lives in a structured directory tree:

```text
app/
  app_services/
    config.json
    auth/
      providers.json
    functions/
      myFunction/
        config.json
        source.js
    triggers/
      myTrigger/
        config.json
    rules/
      myDatabase/
        myCollection/
          schema.json
          rules.json
    values/
      myValue.json
    environments/
      development.json
      production.json
```

## Generate Initial Config with the CLI

If you have an existing App Services application, export its current config:

```bash
npm install -g atlas-app-services-cli
appservices login --api-key <PUBLIC_KEY> --private-api-key <PRIVATE_KEY>
appservices pull --remote="<App ID>" --local ./app_services
```

Commit the exported config to your GitHub repository.

## Enable GitHub Deployment in the Atlas UI

1. Navigate to **App Services > Deploy > Configuration**
2. Select **GitHub** as the deployment method
3. Connect your GitHub account and authorize Atlas
4. Choose your repository and branch (e.g., `main`)
5. Set the directory where your app config lives
6. Click **Enable Automatic Deployment**

## Manual Trigger via GitHub Actions

For more control, trigger deployments from a GitHub Actions workflow:

```yaml
name: Deploy Atlas App Services

on:
  push:
    branches: [main]
    paths:
      - "app_services/**"

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install App Services CLI
        run: npm install -g atlas-app-services-cli

      - name: Authenticate
        run: |
          appservices login \
            --api-key ${{ secrets.ATLAS_PUBLIC_KEY }} \
            --private-api-key ${{ secrets.ATLAS_PRIVATE_KEY }}

      - name: Deploy to Atlas
        run: |
          appservices push \
            --remote="${{ secrets.ATLAS_APP_ID }}" \
            --local ./app_services \
            --yes
```

Store your Atlas API keys as GitHub Actions secrets.

## Environment-Specific Configuration

Use App Services environments to manage different values for development and production:

```json
{
  "values": {
    "API_BASE_URL": "https://api.production.example.com"
  }
}
```

Reference values in functions:

```javascript
const baseUrl = context.values.get("API_BASE_URL")
```

Switch between environments when pushing:

```bash
appservices push --remote="<App ID>" --environment production
```

## Review Deployment History

Atlas tracks every deployment with a diff of what changed. View deployment history in **App Services > Deploy > History** or via the CLI:

```bash
appservices deployments list --remote="<App ID>"
```

Roll back to a previous deployment if a release introduces issues.

## Summary

GitHub integration for Atlas App Services enables GitOps-style deployments where every configuration change flows through pull requests and code review before reaching production. Export your existing config with the CLI, commit it to a repository, and either use automatic deployment from the UI or a GitHub Actions workflow for more control. Use environments to manage deployment-specific values cleanly.
