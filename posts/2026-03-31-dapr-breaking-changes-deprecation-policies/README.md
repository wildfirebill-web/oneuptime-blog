# How to Understand Dapr Breaking Changes and Deprecation Policies

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Breaking Change, Deprecation, Upgrade, Policy

Description: Navigate Dapr's breaking change and deprecation policies to plan upgrades, identify affected APIs and components, and migrate before removal deadlines.

---

Dapr follows a structured policy for deprecating and removing features. Understanding this policy lets you plan migrations proactively rather than discovering breaking changes during an upgrade.

## Dapr's Deprecation Policy

Dapr's general policy for stable APIs:
- Deprecation is announced in the release notes for version N
- The deprecated feature still works for at least 2 minor releases (N+1, N+2)
- Removal happens no earlier than version N+3

For preview features, the timeline is shorter - they may change in any release.

## Finding Breaking Changes

The release notes for each version include a dedicated breaking changes section:

```bash
# For Dapr 1.14
# https://github.com/dapr/dapr/releases/tag/v1.14.0
# Look for the "Breaking Changes" section

# Subscribe to release notifications
# GitHub -> dapr/dapr -> Watch -> Releases only
```

## Component API Deprecations

Dapr components (state stores, pub/sub, bindings) sometimes deprecate metadata fields:

```yaml
# Old (deprecated) configuration
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
spec:
  type: state.redis
  version: v1
  metadata:
  - name: redisHost  # Deprecated in favor of "host" in newer versions
    value: redis:6379
```

Check component-specific migration guides when upgrading:
- https://docs.dapr.io/reference/components-reference/

## SDK Breaking Changes

SDK major versions (1.x -> 2.x) may include breaking API changes. Check the changelog:

```bash
# Python SDK changelog
# https://github.com/dapr/python-sdk/blob/main/CHANGELOG.md

# JavaScript SDK
# https://github.com/dapr/js-sdk/blob/main/CHANGELOG.md

# Check what version you're on
pip show dapr | grep Version
```

Common SDK breaking changes include:
- Method signature changes (e.g., parameter order)
- Module path renames
- Response object field renames

## Detecting Deprecation Warnings at Runtime

Dapr logs deprecation warnings at startup when deprecated configuration is detected:

```bash
# Check Dapr sidecar logs for deprecation warnings
kubectl logs -n default -l app=myapp -c daprd | grep -i "deprecat\|WARN"

# Check Dapr operator logs
kubectl logs -n dapr-system -l app=dapr-operator | grep -i "deprecat"
```

## Practical Migration Example

If Dapr deprecates an actor reminder API, here is a typical migration:

```python
# Before (deprecated)
from dapr.actor import Actor
class MyActor(Actor):
    async def register_reminder(self):
        await self.register_actor_reminder(
            reminder_name="check",
            due_time=timedelta(seconds=10),
            period=timedelta(minutes=1),
            state=b"reminder-data"
        )
```

```python
# After (new API using Scheduler)
from dapr.actor import Actor
class MyActor(Actor):
    async def register_reminder(self):
        await self.create_reminder(
            name="check",
            due_time=timedelta(seconds=10),
            period=timedelta(minutes=1),
            data=b"reminder-data",
            ttl=timedelta(hours=24)
        )
```

## Building a Deprecation Tracking Process

Add deprecation tracking to your upgrade process:

```bash
#!/bin/bash
# deprecation-check.sh - Run before each Dapr upgrade
CURRENT=$(dapr version -k --output json | python3 -c "import sys,json; print(json.load(sys.stdin)['Runtime version'])")
TARGET=$1

echo "Checking breaking changes from $CURRENT to $TARGET"
echo "Release notes: https://github.com/dapr/dapr/releases/tag/v$TARGET"
echo ""
echo "Scanning for deprecated component configurations..."
kubectl get components -A -o yaml | grep -i "deprecat" || echo "None found in cluster"
```

## Summary

Dapr's deprecation policy provides at least 2 minor release cycles between announcement and removal for stable APIs. Breaking changes and deprecations are documented in each version's release notes and can be detected at runtime through Dapr sidecar log warnings. Building a pre-upgrade deprecation check into your deployment process prevents surprises when moving between minor versions.
