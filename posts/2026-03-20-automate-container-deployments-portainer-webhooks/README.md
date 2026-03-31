# How to Automate Container Deployments with Portainer Webhooks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Webhook, Automation, CI/CD, Container Deployment

Description: Learn how to use Portainer service and container webhooks to automate image updates and container redeployments.

## Types of Portainer Webhooks

Portainer supports webhooks at multiple levels:

| Webhook Type | Target | Action |
|-------------|--------|--------|
| Stack webhook | Docker Compose stack | Redeploy entire stack |
| Service webhook | Docker Swarm service | Update service image |
| Container webhook | Standalone container | Recreate container with latest image |

## Creating a Container Webhook

1. Go to your Docker environment in Portainer.
2. Navigate to **Containers** and click on a container.
3. In the container detail view, find the **Webhooks** section.
4. Toggle **Enable webhook** to On.
5. Copy the webhook URL.

## Creating a Service Webhook (Swarm)

1. Navigate to **Services** in your Swarm environment.
2. Click on a service.
3. Find the **Webhooks** tab.
4. Enable and copy the webhook URL.

## Triggering Webhooks

```bash
# Basic webhook trigger

curl -X POST "https://portainer.mycompany.com/api/webhooks/token123"

# With a specific image tag (recommended for traceability)
curl -X POST "https://portainer.mycompany.com/api/webhooks/token123?tag=v2.1.0"

# Verify success (expects 204 No Content)
HTTP_STATUS=$(curl -s -w "%{http_code}" -o /dev/null \
  -X POST "https://portainer.mycompany.com/api/webhooks/token123?tag=v2.1.0")
echo "Response: $HTTP_STATUS"
```

## Triggering from a Registry Webhook

Many container registries can call a URL when an image is pushed. Configure your registry to call Portainer's webhook:

### Docker Hub Webhook

1. In Docker Hub, go to your repository.
2. Click **Webhooks** and add your Portainer webhook URL.
3. Docker Hub calls it after every successful push.

### Harbor Webhook

```bash
# In Harbor UI: Project > Webhooks
# Type: HTTP
# Endpoint URL: https://portainer.mycompany.com/api/webhooks/token123
# Event types: Push artifact
```

## Chaining Webhooks: CI → Registry → Portainer

```bash
#!/bin/bash
# Full automated pipeline: Build > Push > Deploy

set -e

IMAGE_TAG="$1"
REGISTRY="registry.mycompany.com"
IMAGE="${REGISTRY}/myapp:${IMAGE_TAG}"

echo "Building image: ${IMAGE}"
docker build -t "${IMAGE}" .

echo "Pushing to registry..."
docker push "${IMAGE}"

echo "Triggering Portainer deployment..."
HTTP_STATUS=$(curl -s -w "%{http_code}" -o /dev/null \
  -X POST "${PORTAINER_WEBHOOK_URL}?tag=${IMAGE_TAG}")

if [ "${HTTP_STATUS}" = "204" ]; then
  echo "Deployment of ${IMAGE_TAG} triggered successfully"
else
  echo "Deployment failed: HTTP ${HTTP_STATUS}"
  exit 1
fi
```

## Monitoring Webhook Deployments

```bash
# After triggering a webhook, check container status
sleep 30  # Wait for deployment to complete

CONTAINER_STATUS=$(curl -s \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/containers/json" \
  -H "Authorization: Bearer ${API_TOKEN}" | \
  jq -r '.[] | select(.Names[0] == "/myapp") | .Status')

echo "Container status: ${CONTAINER_STATUS}"
```

## Webhook Failure Handling

```bash
#!/bin/bash
# Retry webhook trigger with exponential backoff

WEBHOOK_URL="$1"
MAX_RETRIES=3
RETRY_DELAY=5

for i in $(seq 1 $MAX_RETRIES); do
  HTTP_STATUS=$(curl -s -w "%{http_code}" -o /dev/null \
    -X POST "${WEBHOOK_URL}")

  if [ "${HTTP_STATUS}" = "204" ]; then
    echo "Webhook triggered successfully (attempt ${i})"
    exit 0
  fi

  echo "Attempt ${i} failed: HTTP ${HTTP_STATUS}"
  [ $i -lt $MAX_RETRIES ] && sleep $((RETRY_DELAY * i))
done

echo "Webhook failed after ${MAX_RETRIES} attempts"
exit 1
```

## Conclusion

Portainer webhooks are the simplest and most flexible way to automate container deployments. Whether triggered by a CI pipeline, a registry push event, or a cron job, webhooks provide a stateless, token-secured deployment endpoint that works with any tool that can make HTTP requests.
