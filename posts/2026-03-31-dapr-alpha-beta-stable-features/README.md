# How to Understand Dapr Alpha vs Beta vs Stable Features

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Feature Flag, Alpha, Beta, Stability, Production

Description: Learn the difference between Dapr Alpha, Beta, and Stable features, how to enable preview features, and which stability level is appropriate for production use.

---

## Feature Stability in Dapr

Dapr classifies runtime features into three stability tiers. This applies not just to components but to APIs, building block capabilities, and configuration options:

- **Alpha** - Early preview, may change or be removed in any release
- **Beta** - Approaching stability, changes will have migration paths
- **Stable** - Fully supported with backward compatibility guarantees

## Enabling Alpha Features

Alpha features must be explicitly opt-in via the Dapr Configuration resource:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: appconfig
spec:
  features:
    - name: HotReload
      enabled: true
    - name: AppHealthCheck
      enabled: true
```

Apply the configuration and reference it in your deployment:

```yaml
annotations:
  dapr.io/config: appconfig
```

## Checking Available Feature Flags

List all feature flags for your Dapr runtime version:

```bash
# Query the Dapr operator for available feature flags
kubectl get configuration appconfig -n default -o yaml

# Check which features are enabled on a running sidecar
curl http://localhost:3500/v1.0/metadata | jq '.runtimeMetadata.enabledFeatures'
```

## Beta Features Example: Workflow API

The Dapr Workflow building block was Beta in Dapr 1.11 before becoming Stable in 1.13. During the Beta period, the API used alpha versioning:

```bash
# Beta workflow API (alpha1 endpoint)
POST /v1.0-alpha1/workflow/dapr/{workflowName}/start

# After Stable promotion, moved to:
POST /v1.0/workflow/dapr/{workflowName}/start
```

Code that called the alpha endpoint needed to be updated when the feature graduated.

## Stable Features: Standard Building Blocks

These building blocks are Stable in current Dapr releases:

| Feature | Status |
|---------|--------|
| Service Invocation | Stable |
| State Management | Stable |
| Pub/Sub Messaging | Stable |
| Bindings | Stable |
| Actors | Stable |
| Secrets | Stable |
| Configuration | Stable |
| Workflow | Stable (since 1.13) |

## Production Guidance

Use this checklist before adopting a feature:

```yaml
feature_adoption_checklist:
  - check: "Is the feature listed as Stable in the current release notes?"
  - check: "Does the feature have a documented migration path from Beta?"
  - check: "Are there known issues filed against this feature in GitHub?"
  - check: "Is there a conformance test for the component used with this feature?"
  - check: "Does your team have rollback procedures if the feature causes issues?"
```

## Tracking Feature Graduation

Follow feature graduation via GitHub milestones:

```bash
# Search for graduation issues
open "https://github.com/dapr/dapr/issues?q=label%3Afeature-graduation"
```

Subscribe to the Dapr blog and Discord to receive announcements when major features graduate to Stable.

## Summary

Dapr features progress through Alpha, Beta, and Stable tiers. Alpha features require explicit opt-in via Configuration resources and should only be used in development. Beta features are appropriate for non-critical workloads with a monitored rollout. Stable features carry backward compatibility guarantees and are safe for production. Always verify a feature's stability tier in the release notes before adopting it in production.
