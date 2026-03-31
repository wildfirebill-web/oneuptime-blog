# How to Create Dapr Development Standards for Your Team

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Standard, Best Practice, Team, Governance

Description: Learn how to define Dapr development standards covering naming conventions, component configuration, error handling, and observability to ensure consistency across your team.

---

As Dapr adoption grows across a team, inconsistent patterns accumulate - different app ID conventions, ad-hoc component names, missing resiliency policies. Defining standards early prevents technical debt.

## App ID Naming Convention

Dapr app IDs are used in service invocation, distributed traces, and logs. Use a consistent format:

```markdown
Convention: {team}-{service}-{env}
Examples:
  payments-processor-prod
  inventory-api-staging
  notifications-worker-prod
```

Enforce in Kubernetes via a validating webhook or a CI lint step:

```bash
# CI check: validate app-id matches convention
APP_ID=$(kubectl get deploy $SERVICE -o jsonpath='{.spec.template.metadata.annotations.dapr\.io/app-id}')
if [[ ! "$APP_ID" =~ ^[a-z]+-[a-z]+-[a-z]+$ ]]; then
  echo "ERROR: app-id '$APP_ID' does not match {team}-{service}-{env} convention"
  exit 1
fi
```

## Component Naming Convention

```markdown
Convention: {type}-{env}
Examples:
  statestore-prod
  pubsub-prod
  secrets-prod
```

For environment-specific components, use namespaces rather than name suffixes when possible.

## Required Annotations Checklist

Every Dapr-enabled deployment must include:

```yaml
annotations:
  dapr.io/enabled: "true"
  dapr.io/app-id: "{team}-{service}"
  dapr.io/app-port: "{port}"
  dapr.io/config: "dapr-config"          # points to standard Configuration
  dapr.io/log-level: "info"              # never "debug" in production
  dapr.io/enable-api-logging: "false"    # only true for debugging
```

## Standard Resiliency Policy

Apply a baseline resiliency policy to all services:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Resiliency
metadata:
  name: standard-resiliency
spec:
  policies:
    timeouts:
      default: 10s
    retries:
      default:
        policy: exponential
        maxRetries: 3
        maxInterval: 10s
    circuitBreakers:
      default:
        maxRequests: 1
        timeout: 30s
        trip: consecutiveFailures >= 5
  targets:
    apps:
      {}   # applies to all apps via default policies
```

## Error Handling Standards

Standardize HTTP status codes returned from app endpoints:

```markdown
Dapr sidecar interprets response codes:
- 2xx: Message/call processed successfully
- 404: Resource not found (do NOT retry)
- 429: Too many requests (retry with backoff)
- 500: Internal error (retry if policy configured)

Standard: Always return 200 for pub/sub handlers
unless you intentionally want the message retried (return 500)
or dropped (return 404).
```

## Observability Standards

Require all services to expose a `/health` endpoint and configure it:

```yaml
annotations:
  dapr.io/app-health-check-path: "/health"
  dapr.io/app-health-probe-interval: "30"
  dapr.io/app-health-probe-timeout: "5"
  dapr.io/app-health-threshold: "3"
```

## Standards Document Template

Keep your standards in a team wiki with version history:

```markdown
# Dapr Standards - Team Payments
Version: 1.2
Last updated: 2026-03-31

## Naming Conventions
...

## Required Component Configuration
...

## Resiliency Policy
...

## Prohibited Patterns
- Do not use dapr.io/log-level: "debug" in production
- Do not use wildcard scopes on components
- Do not store credentials in component YAML values (use secretKeyRef)
```

## Summary

Dapr development standards should cover app ID naming, component naming, required annotations, a baseline resiliency policy, error handling conventions, and observability requirements. Store standards in a versioned wiki document and enforce them through CI checks and code review.
