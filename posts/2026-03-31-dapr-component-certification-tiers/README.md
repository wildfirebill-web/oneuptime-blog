# How to Understand Dapr Component Certification Tiers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Component, Certification, Stability, Production

Description: Learn what Dapr component certification tiers mean, how they affect production readiness, and how to choose between Alpha, Beta, and Stable components for your services.

---

## What Are Dapr Component Certification Tiers?

Dapr components (state stores, pub/sub brokers, bindings, secret stores) go through a certification process before being marked stable. The certification tier tells you how well-tested a component is and what level of production support you can expect.

The three tiers are:
- **Alpha** - Community contributed, basic testing
- **Beta** - Broader testing, approaching stability
- **Stable** - Fully certified, production recommended

## Alpha Tier

Alpha components have been contributed to the project but have not yet passed the full conformance test suite. They may:
- Have incomplete feature support
- Lack proper error handling
- Change behavior in future versions

Example alpha components: some newer cloud-native bindings

```yaml
spec:
  type: state.in-memory    # Often alpha tier for non-production use
  version: v1
```

Use alpha components only in development and testing environments.

## Beta Tier

Beta components have passed the conformance test suite and are actively used by the community, but may still have edge cases. They receive:
- Active bug fixes
- Migration paths for breaking changes
- Broader community validation

```yaml
spec:
  type: state.postgresql
  version: v1
```

Beta components are appropriate for non-critical workloads.

## Stable Tier

Stable components have passed extended conformance testing and have demonstrated production usage at scale. They receive:
- Long-term support
- Guaranteed migration paths
- Security patch backporting

```yaml
spec:
  type: state.redis
  version: v1
```

Common stable components:
- `state.redis`
- `pubsub.redis`
- `pubsub.kafka`
- `state.azure.cosmosdb`
- `secretstores.kubernetes`

## Checking Component Certification Status

Visit the components-contrib repository to see certification status:

```bash
open https://github.com/dapr/components-contrib/blob/master/README.md
```

Or check the Dapr docs:

```bash
# Component status table
open https://docs.dapr.io/reference/components-reference/supported-state-stores/
```

Each component page shows the certification level in a badge.

## Conformance Tests for Components

The certification process uses conformance tests. You can run them yourself to validate a component:

```bash
git clone https://github.com/dapr/components-contrib.git
cd components-contrib

# Run Redis state store conformance tests
DAPR_TEST_REDIS_HOST=localhost:6379 \
  go test ./tests/conformance/... -run TestStateConformanceRedis -v
```

## Choosing Components for Production

Use this decision guide:

```
Is the component Stable?
  Yes -> Safe to use in production
  No -> Is it Beta?
    Yes -> Acceptable for non-critical services with monitoring
    No (Alpha) -> Dev/test only, never production
```

## Summary

Dapr component certification tiers (Alpha, Beta, Stable) indicate how well-tested and production-ready a component is. Stable components have passed the full conformance test suite and are recommended for production. Before adopting a component, check its certification status in the Dapr documentation and choose Stable components for critical services.
