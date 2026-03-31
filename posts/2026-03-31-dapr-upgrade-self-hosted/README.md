# How to Upgrade Dapr in Self-Hosted Mode

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Upgrade, Self-Hosted, CLI, Maintenance

Description: Step-by-step guide to upgrading Dapr in self-hosted mode using the Dapr CLI, including how to handle breaking changes and verify a successful upgrade.

---

## Overview

Keeping Dapr up to date in self-hosted mode involves upgrading the CLI, the runtime, and any components that have changed between versions. The Dapr CLI provides a streamlined upgrade path.

## Step 1: Check Current Versions

Before upgrading, document your current versions:

```bash
dapr --version
# CLI version: 1.13.0
# Runtime version: 1.13.0
```

Check running containers:

```bash
docker ps --format "table {{.Image}}\t{{.Names}}"
```

## Step 2: Upgrade the Dapr CLI

```bash
# macOS / Linux
wget -q https://raw.githubusercontent.com/dapr/cli/master/install/install.sh -O - | /bin/bash

# Windows (PowerShell)
powershell -Command "iwr -useb https://raw.githubusercontent.com/dapr/cli/master/install/install.ps1 | iex"

# Verify
dapr --version
```

## Step 3: Uninstall the Current Dapr Runtime

```bash
dapr uninstall
```

This stops and removes the Redis and Zipkin containers but does NOT remove your component configuration files in `~/.dapr/components/`.

Verify cleanup:

```bash
docker ps -a | grep dapr
# Should show nothing
```

## Step 4: Initialize the New Dapr Runtime

```bash
dapr init
# Or specify an exact version
dapr init --runtime-version 1.14.0
```

Confirm the upgrade:

```bash
dapr --version
# CLI version: 1.14.0
# Runtime version: 1.14.0
```

## Step 5: Review Breaking Changes

Check the Dapr release notes for your target version:

```bash
# Review the changelog
curl -s https://api.github.com/repos/dapr/dapr/releases/latest | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['body'][:2000])"
```

Common breaking changes to watch for:
- Component API version bumps (e.g., `v1` to `v2`)
- Renamed configuration fields
- Deprecated APIs

## Step 6: Update Component Files

If components changed schema, update `~/.dapr/components/`:

```yaml
# Before (v1.13)
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
spec:
  type: state.redis
  version: v1
  metadata:
    - name: redisHost
      value: "localhost:6379"

# After (v1.14) - check for new required fields
    - name: enableTLS
      value: "false"
```

## Step 7: Restart Applications

```bash
# Stop all running dapr apps
dapr list
dapr stop --app-id my-app

# Restart with new runtime
dapr run --app-id my-app --app-port 8080 node app.js
```

## Summary

Upgrading Dapr in self-hosted mode is a three-step process: upgrade the CLI, uninstall the old runtime, and reinitialize with the new version. Your component configuration files are preserved across upgrades, but you should always review release notes for breaking changes. Testing the upgraded runtime with a simple `dapr run` command confirms a successful upgrade before restarting all services.
