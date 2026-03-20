# How to Configure Health Check Timeout in Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Health Checks, Timeout

Description: Learn how to configure the health check timeout in Podman to define how long a health check command can run before being considered failed.

---

> The health check timeout sets the maximum duration a health check command can take before Podman treats it as a failure.

When a health check command takes too long, it usually indicates a problem with the application. By setting an appropriate timeout, you ensure that hung or slow health checks are caught and reported as failures rather than blocking indefinitely.

---

## Setting the Health Check Timeout

Use the `--health-timeout` flag to specify the maximum allowed duration:

```bash
# Allow up to 10 seconds for the health check to complete

podman run -d \
  --name my-service \
  --health-cmd "curl -f http://localhost:8080/health || exit 1" \
  --health-timeout 10s \
  --health-interval 30s \
  my-service:latest
```

## Understanding Timeout Behavior

When the timeout is exceeded, Podman kills the health check process and records the check as failed:

```bash
# Short timeout for fast-responding services
podman run -d --name fast-api \
  --health-cmd "curl -f http://localhost:3000/ping || exit 1" \
  --health-timeout 3s \
  --health-interval 15s \
  --health-retries 3 \
  fast-api:latest

# Longer timeout for services with slow startup queries
podman run -d --name db-service \
  --health-cmd "pg_isready -U postgres -d mydb || exit 1" \
  --health-timeout 30s \
  --health-interval 60s \
  --health-retries 5 \
  postgres:15
```

## Timeout in a Containerfile

Define the timeout directly in your Containerfile:

```dockerfile
FROM golang:1.21-alpine AS builder
WORKDIR /app
COPY . .
RUN go build -o server .

FROM alpine:3.18
COPY --from=builder /app/server /server

# Set a 5-second timeout for the health check
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:8080/ready || exit 1

CMD ["/server"]
```

## Combining Timeout with Other Options

```bash
# Full health check configuration with timeout
podman run -d \
  --name production-app \
  --health-cmd "curl -sf http://localhost:8080/health || exit 1" \
  --health-interval 20s \
  --health-timeout 5s \
  --health-retries 3 \
  --health-start-period 30s \
  production-app:latest
```

## Inspecting the Configured Timeout

```bash
# View the timeout value for a running container
podman inspect --format='{{.Config.Healthcheck.Timeout}}' my-service

# Check if recent health checks timed out
podman inspect --format='{{json .State.Health}}' my-service | python3 -m json.tool
```

## Summary

The `--health-timeout` flag defines how long Podman waits for a health check command to complete before marking it as failed. Set shorter timeouts for services expected to respond quickly, and longer timeouts for services that may need more time to respond. The default timeout is 30 seconds, but most applications benefit from a tighter value.
