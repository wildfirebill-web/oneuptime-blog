# How to Use Podman in Buildkite Pipelines

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Container, DevOps, Buildkite, CI/CD, Pipeline

Description: Learn how to configure and use Podman in Buildkite pipelines for building, testing, and deploying container images with self-hosted agents.

---

> Buildkite's agent-based architecture makes it straightforward to use Podman since you control the agent environment and can install Podman directly.

Buildkite uses self-hosted agents, which means you have full control over the build environment. This makes Podman integration simple -- install Podman on your agents and use it directly in your pipeline steps. This guide covers practical patterns for using Podman in Buildkite pipelines for container workflows.

---

## Setting Up Buildkite Agents with Podman

Install Podman on your Buildkite agent machines.

```bash
#!/bin/bash
# Install Podman on a Buildkite agent running Ubuntu

# Run this as part of your agent provisioning

# Install Podman
sudo apt-get update
sudo apt-get install -y podman fuse-overlayfs

# Configure storage for the buildkite-agent user
sudo -u buildkite-agent mkdir -p /home/buildkite-agent/.config/containers
sudo -u buildkite-agent tee /home/buildkite-agent/.config/containers/storage.conf << 'EOF'
[storage]
driver = "overlay"

[storage.options.overlay]
mount_program = "/usr/bin/fuse-overlayfs"
EOF

# Verify Podman works for the buildkite-agent user
sudo -u buildkite-agent podman info
sudo -u buildkite-agent podman --version

# Add a tag to the agent so pipelines can target Podman-enabled agents
# In /etc/buildkite-agent/buildkite-agent.cfg:
# tags="podman=true"
```

## Basic Buildkite Pipeline with Podman

Create a pipeline that builds and tests a container image.

```yaml
# .buildkite/pipeline.yml
# Basic Podman build and test pipeline
steps:
  # Verify Podman is available on the agent
  - label: ":podman: Check Podman"
    command: podman --version
    agents:
      podman: "true"

  # Build the container image
  - label: ":building_construction: Build Image"
    commands:
      - podman build
          --tag myapp:${BUILDKITE_BUILD_NUMBER}
          --tag myapp:${BUILDKITE_COMMIT}
          .
    agents:
      podman: "true"

  # Run the test suite
  - label: ":test_tube: Run Tests"
    depends_on: ":building_construction: Build Image"
    commands:
      - podman run --rm myapp:${BUILDKITE_COMMIT} npm test
    agents:
      podman: "true"
```

## Building and Pushing Images to a Registry

Push images to a container registry after successful builds.

```yaml
# .buildkite/pipeline.yml
# Build and push pipeline with registry authentication
steps:
  - label: ":building_construction: Build"
    commands:
      - podman build
          -t ${REGISTRY}/${IMAGE_NAME}:${BUILDKITE_COMMIT}
          -t ${REGISTRY}/${IMAGE_NAME}:latest
          .
    agents:
      podman: "true"

  - label: ":rocket: Push to Registry"
    depends_on: ":building_construction: Build"
    commands:
      # Login using credentials from Buildkite secrets
      - echo "$REGISTRY_PASSWORD" |
          podman login $REGISTRY
            -u "$REGISTRY_USERNAME"
            --password-stdin

      # Push both tags
      - podman push ${REGISTRY}/${IMAGE_NAME}:${BUILDKITE_COMMIT}
      - podman push ${REGISTRY}/${IMAGE_NAME}:latest
    agents:
      podman: "true"
    # Only push on the main branch
    branches: "main"
```

## Integration Testing with Podman

Run multi-container integration tests in Buildkite.

```yaml
# .buildkite/pipeline.yml
# Integration testing with Podman
steps:
  - label: ":building_construction: Build"
    commands:
      - podman build -t myapp:test .
    agents:
      podman: "true"

  - label: ":database: Integration Tests"
    depends_on: ":building_construction: Build"
    commands:
      # Create a network for the test services
      - podman network create bk-test-net

      # Start PostgreSQL
      - podman run -d
          --name bk-postgres
          --network bk-test-net
          -e POSTGRES_PASSWORD=testpass
          -e POSTGRES_DB=testdb
          postgres:16-alpine

      # Wait for database
      - sleep 5
      - podman exec bk-postgres pg_isready -U postgres

      # Run integration tests
      - podman run --rm
          --network bk-test-net
          -e DATABASE_URL=postgresql://postgres:testpass@bk-postgres/testdb
          myapp:test npm run test:integration

      # Clean up
      - podman rm -f bk-postgres
      - podman network rm bk-test-net
    agents:
      podman: "true"
```

## Using Buildkite Plugins with Podman

Create a custom Buildkite plugin for common Podman operations.

```yaml
# .buildkite/pipeline.yml
# Using environment hooks for Podman setup
steps:
  - label: ":building_construction: Build and Test"
    commands:
      - podman build -t myapp:${BUILDKITE_COMMIT} .
      - podman run --rm myapp:${BUILDKITE_COMMIT} npm test
    env:
      STORAGE_DRIVER: overlay
    agents:
      podman: "true"
```

```bash
#!/bin/bash
# .buildkite/hooks/pre-command
# Pre-command hook that runs before every build step
# Use this to configure Podman for the build environment

# Clean up any leftover containers from previous builds
podman container prune -f 2>/dev/null || true

# Prune old images to save disk space (keep images from last 24h)
podman image prune -f --filter "until=24h" 2>/dev/null || true

echo "Podman environment ready"
podman info --format 'Storage driver: {{.Store.GraphDriverName}}'
```

## Dynamic Pipeline Generation

Generate pipeline steps dynamically using Podman for complex workflows.

```bash
#!/bin/bash
# .buildkite/scripts/generate-pipeline.sh
# Generate Buildkite pipeline steps based on changed files

cat << 'YAML'
steps:
YAML

# Always build the image
cat << YAML
  - label: ":building_construction: Build Image"
    commands:
      - podman build -t myapp:${BUILDKITE_COMMIT} .
    agents:
      podman: "true"
YAML

# Add test steps based on what changed
if git diff --name-only HEAD~1 | grep -q "^src/"; then
cat << YAML
  - label: ":test_tube: Unit Tests"
    depends_on: ":building_construction: Build Image"
    commands:
      - podman run --rm myapp:${BUILDKITE_COMMIT} npm test
    agents:
      podman: "true"
YAML
fi

if git diff --name-only HEAD~1 | grep -q "^test/integration"; then
cat << YAML
  - label: ":database: Integration Tests"
    depends_on: ":building_construction: Build Image"
    commands:
      - podman network create bk-net
      - podman run -d --name testdb --network bk-net -e POSTGRES_PASSWORD=test postgres:16-alpine
      - sleep 5
      - podman run --rm --network bk-net -e DB_HOST=testdb myapp:${BUILDKITE_COMMIT} npm run test:integration
      - podman rm -f testdb
      - podman network rm bk-net
    agents:
      podman: "true"
YAML
fi
```

```yaml
# .buildkite/pipeline.yml for dynamic generation
steps:
  - label: ":pipeline: Generate Pipeline"
    command: .buildkite/scripts/generate-pipeline.sh | buildkite-agent pipeline upload
```

## Summary

Buildkite's self-hosted agent model gives you full control over the build environment, making Podman integration straightforward. Install Podman on your agents, tag them appropriately, and use Podman commands directly in your pipeline steps. Pre-command hooks help manage cleanup and configuration, while dynamic pipeline generation lets you create complex workflows based on code changes. The agent-based approach means you can optimize your Podman configuration per machine, including storage drivers and caching strategies, for the best possible performance.
