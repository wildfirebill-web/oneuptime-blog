# How to Access the Portainer API Documentation - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, API, Documentation, DevOps, Automation

Description: Learn how to access and navigate the Portainer API documentation, including the built-in Swagger UI, API versioning, and how to explore available endpoints.

## Introduction

Portainer exposes a comprehensive REST API that allows you to automate virtually every operation available in the UI. Before writing API scripts, it is important to know how to find and navigate the API documentation. Portainer ships with a built-in Swagger UI and also publishes documentation online.

## Prerequisites

- Portainer CE or BE installed and running
- Access to the Portainer web interface
- A web browser

## Accessing the Built-in Swagger UI

Portainer includes an interactive Swagger UI accessible from your Portainer instance:

### URL Format

```text
https://portainer.example.com/api/documentation
```

or on a local installation:

```text
http://localhost:9000/api/documentation
```

This opens the Swagger UI directly from your Portainer instance, showing the exact API version your deployment supports.

## Navigating the Swagger UI

The Swagger UI organizes endpoints by **tags** (resource categories):

| Tag | Description |
|-----|-------------|
| `auth` | Authentication (login, logout, token management) |
| `users` | User management |
| `teams` | Team management |
| `endpoints` | Environment management |
| `stacks` | Stack (Compose) management |
| `containers` | Direct Docker container operations |
| `images` | Container image management |
| `volumes` | Volume management |
| `networks` | Network management |
| `registries` | Registry management |
| `kubernetes` | Kubernetes-specific operations |
| `webhooks` | Webhook management |

### Making Test Requests in Swagger UI

1. Open the Swagger UI.
2. Click the **Authorize** button at the top.
3. Enter your JWT token: `Bearer your-jwt-token`
4. Click **Authorize**, then **Close**.
5. Expand any endpoint group (e.g., `auth`).
6. Click an individual endpoint (e.g., `POST /api/auth`).
7. Click **Try it out**.
8. Fill in the request body.
9. Click **Execute**.
10. View the response below.

## Accessing the Official Online Documentation

Portainer publishes API documentation online:

```text
https://app.swaggerhub.com/apis/portainer/portainer-ce/2.21.0
```

The version number in the URL corresponds to your Portainer version. Check your Portainer version first:

```bash
# Check your Portainer version

curl -s https://portainer.example.com/api/system/status | jq .Version
```

Then access the matching documentation:

```bash
PORTAINER_VERSION="2.21.0"
echo "API Docs: https://app.swaggerhub.com/apis/portainer/portainer-ce/${PORTAINER_VERSION}"
```

## Understanding API Versioning

Portainer's API does not use separate version prefixes (like `/v1/`, `/v2/`). All endpoints are under `/api/`. Breaking changes are documented in release notes.

Key API URL patterns:

```bash
# Global operations
POST   /api/auth                         # Get JWT token
GET    /api/users                         # List users
GET    /api/endpoints                     # List environments

# Endpoint-scoped Docker operations
GET    /api/endpoints/{id}/docker/containers/json   # List containers
POST   /api/endpoints/{id}/docker/containers/create # Create container
GET    /api/endpoints/{id}/docker/volumes           # List volumes

# Endpoint-scoped Kubernetes operations
GET    /api/endpoints/{id}/kubernetes/api/v1/namespaces
GET    /api/endpoints/{id}/kubernetes/helm

# Stack operations
GET    /api/stacks
POST   /api/stacks/create/standalone/string
```

## Downloading the OpenAPI Spec

Download the raw OpenAPI spec for use with code generators:

```bash
# Download the Portainer OpenAPI specification
curl -s https://portainer.example.com/api/documentation/json -o portainer-openapi.json

# Pretty print to review
cat portainer-openapi.json | jq '{
  title: .info.title,
  version: .info.version,
  paths: (.paths | keys | length)
}'
```

## Generating Client SDKs from the API Spec

Use OpenAPI Generator to create a client library:

```bash
# Install OpenAPI Generator
npm install -g @openapitools/openapi-generator-cli

# Generate a Python client
openapi-generator-cli generate \
  -i portainer-openapi.json \
  -g python \
  -o portainer-python-client \
  --package-name portainer_client

# Generate a JavaScript/TypeScript client
openapi-generator-cli generate \
  -i portainer-openapi.json \
  -g typescript-fetch \
  -o portainer-ts-client
```

## Using the API with curl Examples

```bash
# Quick reference: common API calls

# 1. Authenticate
curl -X POST https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"password"}'

# 2. List environments
curl -H "Authorization: Bearer TOKEN" \
  https://portainer.example.com/api/endpoints

# 3. List stacks
curl -H "Authorization: Bearer TOKEN" \
  https://portainer.example.com/api/stacks

# 4. List users
curl -H "Authorization: Bearer TOKEN" \
  https://portainer.example.com/api/users
```

## Conclusion

The Portainer API documentation is accessible through the built-in Swagger UI at `/api/documentation` on your Portainer instance, as well as online via SwaggerHub. Use the Swagger UI to explore and test endpoints interactively, download the OpenAPI spec for client generation, and always match documentation to your specific Portainer version for accuracy.
