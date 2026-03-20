# How to Access the Portainer API Documentation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, API, Swagger, REST API, Automation

Description: Learn how to find and navigate the Portainer API documentation to start automating your container management workflows.

## Where to Find the Portainer API Docs

Portainer exposes a Swagger UI for interactive API documentation. There are two ways to access it:

### 1. Built-In Swagger UI

Navigate to `http(s)://<your-portainer-url>/api/documentation` in your browser. This serves the interactive Swagger UI where you can browse all endpoints and make test requests.

### 2. Official Online Documentation

The latest Portainer API documentation is available at:
```text
https://app.swaggerhub.com/apis/portainer/portainer-ce/
```

## Exploring the Swagger UI

The Swagger UI groups endpoints by resource type:

- **Auth** - JWT authentication
- **Users** - User management
- **Teams** - Team management
- **Endpoints** - Environment management
- **Stacks** - Stack deployment
- **Containers** - Container operations
- **Images** - Image management
- **Registries** - Registry management
- **Settings** - Global settings

## Authenticating in the Swagger UI

To make API calls directly from Swagger:

1. First, get a JWT token:

```bash
# Get a JWT token from the API

curl -X POST "http://localhost:9000/api/auth" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "admin",
    "password": "yourpassword"
  }'
# Response: {"jwt": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."}
```

2. In Swagger UI, click **Authorize**.
3. Enter `Bearer <your-jwt-token>`.
4. Click **Authorize**.

You can now make authenticated API calls directly from the Swagger interface.

## API Base URL

All API endpoints are under:
```text
https://<your-portainer-host>/api/
```

## Key API Endpoints Quick Reference

```text
GET    /api/endpoints          - List all environments
GET    /api/endpoints/{id}     - Get environment details
GET    /api/stacks             - List all stacks
POST   /api/stacks             - Create a new stack
PUT    /api/stacks/{id}        - Update a stack
DELETE /api/stacks/{id}        - Delete a stack
GET    /api/users              - List users
POST   /api/users              - Create a user
GET    /api/registries         - List registries
```

## Downloading the OpenAPI Spec

```bash
# Download the OpenAPI spec for code generation or testing
curl -o portainer-openapi.json \
  http://localhost:9000/api/documentation/swagger.json

# Use with OpenAPI generators
openapi-generator-cli generate \
  -i portainer-openapi.json \
  -g python \
  -o ./portainer-client
```

## Version-Specific Documentation

Portainer API versioning follows the Portainer release version. Ensure you're reading docs for your installed version:

```bash
# Check Portainer version
curl http://localhost:9000/api/system/status | jq '.Version'
```

## Conclusion

The Portainer API Swagger documentation is your primary reference for automation. Start by exploring the built-in Swagger UI at `/api/documentation`, then use the JWT authentication flow to make live API calls and test your scripts.
