# How to Use the Portainer API as a Docker API Gateway

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, API, Docker API, Gateway, Security

Description: Learn how to use Portainer as a secure gateway to the Docker API, adding authentication and RBAC without exposing the Docker socket directly.

## Why Use Portainer as a Docker API Gateway?

Exposing the Docker socket directly (`/var/run/docker.sock`) is a significant security risk - anyone with socket access has root-equivalent access to the host. Portainer acts as a controlled proxy:

- **Authentication**: All requests require a valid Portainer token.
- **Audit logging**: Every API call is logged.
- **RBAC**: Users only access environments they're authorized for.
- **TLS**: Portainer handles TLS termination.

## Docker API Proxy Endpoint

Portainer proxies all Docker API calls through:
```text
/api/endpoints/{endpointId}/docker/{docker-api-path}
```

This mirrors the Docker API almost exactly, allowing existing Docker client scripts to work with minimal changes.

## Using Portainer as a Docker Host

You can configure your Docker CLI to use Portainer as the Docker host:

```bash
# Set Portainer as the Docker host via environment variable

export DOCKER_HOST="tcp://portainer.mycompany.com:9000"

# However, standard Docker CLI doesn't support Bearer token auth natively
# Use a wrapper script instead
```

## Wrapper Script: Docker Commands via Portainer API

```bash
#!/bin/bash
# portainer-docker.sh - Wrapper to use Docker commands via Portainer API

PORTAINER_URL="https://portainer.mycompany.com"
API_TOKEN="${PORTAINER_API_TOKEN}"
ENDPOINT_ID="${PORTAINER_ENDPOINT_ID:-1}"

docker_api() {
  local method="${1}"
  local path="${2}"
  shift 2

  curl -s -X "${method}" \
    "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker${path}" \
    -H "Authorization: Bearer ${API_TOKEN}" \
    "$@"
}

case "$1" in
  ps)
    docker_api GET "/containers/json${2:+?all=true}" | \
      jq -r '.[] | [.Id[0:12], .Names[0], .Image, .Status] | @tsv' | \
      column -t
    ;;
  logs)
    docker_api GET "/containers/$2/logs?stdout=true&stderr=true&tail=100"
    ;;
  restart)
    docker_api POST "/containers/$2/restart"
    echo "Restarted container $2"
    ;;
  *)
    echo "Usage: $0 {ps|logs <container>|restart <container>}"
    exit 1
    ;;
esac
```

## Comparing Docker vs. Portainer API Calls

```bash
# Standard Docker API call (requires socket access)
curl --unix-socket /var/run/docker.sock \
  http://localhost/v1.43/containers/json

# Same call via Portainer (adds authentication, logging, RBAC)
curl "https://portainer.mycompany.com/api/endpoints/1/docker/containers/json" \
  -H "Authorization: Bearer ${API_TOKEN}"
```

## Setting Up Role-Based Docker API Access

Create a limited user in Portainer who can only view containers in a specific environment:

```bash
# Create a read-only user
curl -X POST "${PORTAINER_URL}/api/users" \
  -H "Authorization: Bearer ${ADMIN_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"Username": "monitoring-bot", "Password": "...", "Role": 2}'

# Grant read-only access to a specific endpoint
curl -X PUT "${PORTAINER_URL}/api/users/${USER_ID}/permissions" \
  -H "Authorization: Bearer ${ADMIN_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{\"EndpointRestrictions\": [{\"Id\": ${ENDPOINT_ID}, \"Access\": 1}]}"
```

## Benefits Over Direct Docker Socket Exposure

| Feature | Direct Socket | Via Portainer |
|---------|--------------|---------------|
| Authentication | None | JWT/Token |
| Audit logs | None | Full API audit log |
| Per-user access control | None | RBAC per environment |
| TLS | No | Yes |
| Multi-host | No | Yes (multiple endpoints) |

## Conclusion

Using Portainer as a Docker API gateway adds security and accountability to Docker operations. Instead of exposing the Docker socket to multiple users or systems, centralize access through Portainer's authenticated proxy for a more secure architecture.
