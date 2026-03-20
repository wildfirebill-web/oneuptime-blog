# How to Set Up Application Webhooks in Portainer for Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Webhooks, CI/CD, Automation

Description: Learn how to configure application webhooks in Portainer to trigger automated redeployments when a new container image is pushed.

## What Are Application Webhooks?

Portainer can expose a webhook URL for any Kubernetes application. When an external system (like a CI/CD pipeline) calls this URL, Portainer triggers a redeploy - pulling the latest version of the container image. This enables automated deployments without direct Portainer API access.

## Enabling a Webhook in Portainer

1. Select your Kubernetes environment.
2. Go to **Applications** and open an application.
3. Click **Edit** on the application.
4. Scroll to the **Webhooks** section.
5. Toggle **Enable webhook** to On.
6. Copy the generated webhook URL.

The URL looks like:
```text
https://portainer.mycompany.com/api/webhooks/abc123def456...
```

## Triggering the Webhook

Call the webhook URL with an HTTP POST request to trigger a redeploy:

```bash
# Trigger a redeploy via curl

curl -X POST \
  "https://portainer.mycompany.com/api/webhooks/abc123def456..."

# With a specific image tag (Portainer webhook supports tag override)
curl -X POST \
  "https://portainer.mycompany.com/api/webhooks/abc123def456...?tag=v2.1.0"
```

## Integrating with GitHub Actions

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: registry.mycompany.com/my-app:${{ github.sha }}

      - name: Trigger Portainer redeploy
        run: |
          curl -X POST \
            "${{ secrets.PORTAINER_WEBHOOK_URL }}?tag=${{ github.sha }}"
```

## Integrating with GitLab CI

```yaml
# .gitlab-ci.yml
deploy:
  stage: deploy
  script:
    - |
      curl -X POST \
        "${PORTAINER_WEBHOOK_URL}?tag=${CI_COMMIT_SHA}"
  only:
    - main
```

## Webhook Security

Webhook URLs are long random tokens. Additionally:

- Serve Portainer behind HTTPS to prevent token interception.
- Rotate webhooks by disabling and re-enabling them in Portainer.
- Restrict webhook triggering to known CI/CD IP ranges at the firewall level.

```bash
# Test that the webhook is working correctly
curl -v -X POST "https://portainer.mycompany.com/api/webhooks/your-token"
# Expect: 204 No Content on success
```

## Conclusion

Application webhooks in Portainer provide a simple, secure trigger point for CI/CD pipelines. They enable zero-touch deployments where pushing a new image to a registry automatically rolls out to your Kubernetes cluster.
