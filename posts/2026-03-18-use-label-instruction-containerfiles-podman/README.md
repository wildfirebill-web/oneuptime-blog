# How to Use LABEL Instruction in Containerfiles for Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containerfile, LABEL, Container Metadata, DevOps

Description: Learn how to use the LABEL instruction in Containerfiles for Podman to add metadata to your container images, including version info, maintainer details, and custom annotations.

---

> Labels turn your container images from anonymous blobs into well-documented, searchable, and maintainable artifacts that your entire team can understand at a glance.

Container images are the building blocks of modern application deployment. But without proper metadata, they can quickly become opaque and difficult to manage. The LABEL instruction in Containerfiles provides a standardized way to attach metadata to your images, making them easier to organize, filter, and maintain. In this guide, we will explore how to use the LABEL instruction effectively with Podman.

---

## What Is the LABEL Instruction?

The LABEL instruction adds key-value metadata pairs to a container image. These labels do not affect the runtime behavior of the container. Instead, they serve as annotations that tools, registries, and operators can query to understand what an image contains, who built it, and how it should be used.

The basic syntax is straightforward:

```dockerfile
LABEL key="value"
```

You can also set multiple labels on a single line:

```dockerfile
LABEL key1="value1" key2="value2" key3="value3"
```

## Basic LABEL Usage

Let us start with a simple Containerfile that uses labels to document a Node.js application image:

```dockerfile
FROM node:20-alpine

LABEL maintainer="dev-team@example.com"
LABEL version="1.0.0"
LABEL description="A Node.js API server for order processing"

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .

EXPOSE 3000
CMD ["node", "server.js"]
```

Build and inspect this image with Podman:

```bash
podman build -t my-api:1.0.0 .
podman inspect my-api:1.0.0 --format '{{json .Config.Labels}}' | jq
```

The output will display all labels associated with the image as a JSON object, letting you programmatically retrieve any metadata you attached during the build.

## Combining Multiple Labels

Each LABEL instruction creates a new layer in the image. To minimize layers and keep images lean, combine multiple labels into a single instruction:

```dockerfile
FROM python:3.12-slim

LABEL maintainer="platform-team@example.com" \
      version="2.3.1" \
      description="Data processing pipeline service" \
      org.opencontainers.image.source="https://github.com/example/data-pipeline" \
      org.opencontainers.image.licenses="MIT"

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .

CMD ["python", "main.py"]
```

This approach produces a single layer for all your labels, keeping the image size as small as possible.

## OCI Standard Labels

The Open Container Initiative (OCI) defines a set of pre-defined label keys under the `org.opencontainers.image` namespace. Using these standardized keys ensures compatibility with container registries, CI/CD tools, and orchestration platforms:

```dockerfile
FROM golang:1.22-alpine AS builder

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go build -o /server .

FROM alpine:3.19

LABEL org.opencontainers.image.title="Go API Server" \
      org.opencontainers.image.description="REST API server built with Go" \
      org.opencontainers.image.version="3.1.0" \
      org.opencontainers.image.authors="backend-team@example.com" \
      org.opencontainers.image.url="https://example.com/go-api" \
      org.opencontainers.image.source="https://github.com/example/go-api" \
      org.opencontainers.image.documentation="https://docs.example.com/go-api" \
      org.opencontainers.image.vendor="Example Corp" \
      org.opencontainers.image.licenses="Apache-2.0" \
      org.opencontainers.image.created="2026-03-18T00:00:00Z" \
      org.opencontainers.image.revision="abc123def"

COPY --from=builder /server /server
ENTRYPOINT ["/server"]
```

Some of the most commonly used OCI labels include `title`, `description`, `version`, `authors`, `source`, `licenses`, `created`, and `revision`. Adopting these keys makes your images interoperable with a wide ecosystem of tools.

## Dynamic Labels with Build Arguments

You can inject label values at build time using ARG instructions. This is particularly useful in CI/CD pipelines where values like commit hashes and build timestamps are generated dynamically:

```dockerfile
FROM node:20-alpine

ARG BUILD_DATE
ARG GIT_COMMIT
ARG VERSION

LABEL org.opencontainers.image.created="${BUILD_DATE}" \
      org.opencontainers.image.revision="${GIT_COMMIT}" \
      org.opencontainers.image.version="${VERSION}" \
      org.opencontainers.image.title="Payment Service"

WORKDIR /app
COPY . .
RUN npm ci --only=production

CMD ["node", "index.js"]
```

Pass the values when building:

```bash
podman build \
  --build-arg BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ') \
  --build-arg GIT_COMMIT=$(git rev-parse HEAD) \
  --build-arg VERSION=1.4.2 \
  -t payment-service:1.4.2 .
```

This ensures every image carries precise provenance information without hardcoding anything in the Containerfile.

## Querying Labels with Podman

Podman provides several ways to work with labels. You can filter images based on label values, which is invaluable when managing large image registries:

```bash
# List all images with a specific label

podman images --filter label=org.opencontainers.image.vendor="Example Corp"

# Get a specific label value
podman inspect my-api:1.0.0 --format '{{index .Config.Labels "version"}}'

# List all labels on an image
podman inspect my-api:1.0.0 --format '{{range $k, $v := .Config.Labels}}{{$k}}={{$v}}{{println}}{{end}}'
```

You can also use labels when running containers to add runtime-specific metadata:

```bash
podman run -d --label environment=staging --label team=backend my-api:1.0.0
```

Then filter running containers by label:

```bash
podman ps --filter label=environment=staging
```

## Labels for Container Orchestration

When using Podman with systemd or Kubernetes, labels play an important role. For example, you can generate systemd unit files from labeled containers:

```dockerfile
FROM nginx:alpine

LABEL io.containers.autoupdate="registry" \
      io.podman.compose.project="web-stack" \
      environment="production" \
      tier="frontend"

COPY nginx.conf /etc/nginx/nginx.conf
COPY html/ /usr/share/nginx/html/
```

The `io.containers.autoupdate` label tells Podman's auto-update feature to check the registry for newer versions of this image. This is a Podman-specific label that integrates with `podman auto-update`.

## Best Practices

When working with labels, keep these guidelines in mind. First, always use namespaced keys for custom labels to avoid collisions. A reverse domain notation like `com.yourcompany.yourapp.key` works well. Second, prefer OCI standard labels over custom ones when a standard key exists for your use case. Third, combine labels into a single LABEL instruction to reduce image layers. Fourth, use build arguments for values that change between builds, such as timestamps and commit hashes. Fifth, document your labeling conventions so your team applies them consistently. Sixth, avoid storing sensitive information in labels since they are visible to anyone who can inspect the image.

## Conclusion

The LABEL instruction is a simple but powerful tool for making your container images self-documenting. By adopting OCI standard labels, injecting dynamic metadata through build arguments, and leveraging Podman's filtering capabilities, you can build a container image management workflow that scales with your organization. Start by adding a few essential labels to your Containerfiles, then expand your labeling strategy as your needs grow.
