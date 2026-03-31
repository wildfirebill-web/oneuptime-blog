# How to Set Up Automated Testing with Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Automated Testing, CI/CD, Docker, Integration Test, Smoke Tests

Description: Learn how to integrate automated tests into your Portainer deployment workflow using ephemeral test containers and health check verification.

---

Automated testing with Portainer involves running test containers against your deployed stacks, verifying health check endpoints, and integrating test results into your CI/CD pipeline. This guide covers smoke tests, integration tests, and ephemeral test environments.

## Running Tests in an Ephemeral Container

Use a `test` service in your stack that runs and exits - Docker won't keep it running:

```yaml
version: "3.8"

services:
  api:
    image: myregistry.example.com/my-app:latest
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app_net

  db:
    image: postgres:15
    environment:
      POSTGRES_DB: testdb
      POSTGRES_USER: testuser
      POSTGRES_PASSWORD: testpassword
    networks:
      - app_net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U testuser"]
      interval: 5s
      retries: 5

  test-runner:
    image: myregistry.example.com/my-app-tests:latest
    environment:
      API_URL: http://api:3000
      DATABASE_URL: postgresql://testuser:testpassword@db:5432/testdb
    depends_on:
      api:
        condition: service_healthy
      db:
        condition: service_healthy
    networks:
      - app_net
    command: npm test

networks:
  app_net:
```

The `test-runner` service exits with code 0 on success or non-zero on failure. CI reads the exit code to determine pass/fail.

## Smoke Test Script

A simple smoke test that verifies key endpoints after deployment:

```bash
#!/bin/bash
# smoke-test.sh <base-url>

BASE_URL="${1:-http://localhost:3000}"
PASS=0
FAIL=0

check() {
    local name=$1
    local url=$2
    local expected_status=${3:-200}

    status=$(curl -s -o /dev/null -w "%{http_code}" "$url")
    if [ "$status" = "$expected_status" ]; then
        echo "PASS: $name ($status)"
        ((PASS++))
    else
        echo "FAIL: $name (expected $expected_status, got $status)"
        ((FAIL++))
    fi
}

check "Health endpoint"       "$BASE_URL/health"
check "API version"           "$BASE_URL/api/version"
check "Login page"            "$BASE_URL/login"
check "Missing page 404"      "$BASE_URL/does-not-exist" 404

echo ""
echo "Results: $PASS passed, $FAIL failed"
[ $FAIL -eq 0 ] && exit 0 || exit 1
```

## Integration Test Workflow

Run integration tests in CI after deploying to staging:

```bash
#!/bin/bash
set -euo pipefail

# Deploy to staging

curl -fsS -X POST "$PORTAINER_STAGING_WEBHOOK"

# Wait for containers to become healthy
echo "Waiting for deployment..."
sleep 30

# Run smoke tests
./scripts/smoke-test.sh https://staging.example.com

# Run integration tests in a Docker container
docker run --rm \
  --network host \
  -e BASE_URL=https://staging.example.com \
  myregistry.example.com/integration-tests:latest

echo "All tests passed"
```

## Database Migration Tests

Test that database migrations apply cleanly in an ephemeral environment:

```bash
# Run migration tests in isolation
docker run --rm \
  --network my-app_app_net \
  -e DATABASE_URL=postgresql://appuser:apppassword@postgres:5432/appdb \
  myregistry.example.com/my-app:latest \
  sh -c "npm run migrate && npm run migrate:verify"
```

## Portainer API-Based Health Verification

Use the Portainer API to verify all services in a stack are healthy before proceeding:

```bash
#!/bin/bash
PORTAINER_URL="https://portainer.example.com"
STACK_NAME="my-app-staging"

TOKEN=$(curl -s -X POST "$PORTAINER_URL/api/auth" \
  -H "Content-Type: application/json" \
  -d '{"Username":"admin","Password":"adminpassword"}' | jq -r .jwt)

STACK_ID=$(curl -s -H "Authorization: Bearer $TOKEN" \
  "$PORTAINER_URL/api/stacks" | \
  jq -r --arg name "$STACK_NAME" '.[] | select(.Name==$name) | .Id')

# Wait up to 120 seconds for all containers to be healthy
for i in $(seq 1 24); do
  UNHEALTHY=$(curl -s -H "Authorization: Bearer $TOKEN" \
    "$PORTAINER_URL/api/stacks/$STACK_ID/file" | \
    # Check via Docker directly
    docker ps --filter "label=com.docker.compose.project=$STACK_NAME" \
              --filter "health=unhealthy" -q | wc -l)

  if [ "$UNHEALTHY" = "0" ]; then
    echo "All containers healthy"
    break
  fi

  echo "Waiting for containers to become healthy ($i/24)..."
  sleep 5
done
```

## Test Result Reporting

Publish test results as JUnit XML for CI systems to parse:

```bash
# Jest with JUnit reporter
docker run --rm \
  -e CI=true \
  -v "$(pwd)/test-results:/app/test-results" \
  myregistry.example.com/my-app-tests:latest \
  npx jest --reporters=jest-junit --outputFile=test-results/results.xml
```
