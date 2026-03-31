# How to Create Dapr Development Standards for Your Team

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Standards, Best Practice, Team, Governance

Description: Define Dapr development standards covering naming conventions, required annotations, component templates, resiliency policies, and code review checklists for consistent usage.

---

## Why Development Standards Matter

Without standards, each team configures Dapr differently - some use inline secrets, others skip resiliency policies, naming conventions diverge. Standards create consistency, reduce onboarding time, and make incident response faster because configurations are predictable.

## Naming Conventions

Establish clear names for Dapr resources:

```yaml
naming_standards:
  app_id:
    format: "{service-name}"
    examples: ["order-processor", "inventory-api", "notification-worker"]
    rules:
      - "Lowercase, hyphenated"
      - "Match the Kubernetes service name"
      - "No underscores or camelCase"

  component_names:
    format: "{resource-type}"
    examples:
      state_store: "redis-statestore"
      pub_sub: "kafka-pubsub"
      secret_store: "vault-secrets"
      binding: "s3-input-binding"
```

## Required Kubernetes Annotations

Define a mandatory annotation set for all Dapr-enabled services:

```yaml
# Mandatory annotations template
annotations:
  dapr.io/enabled: "true"
  dapr.io/app-id: "{{ service-name }}"     # Must match naming convention
  dapr.io/app-port: "{{ http-port }}"
  dapr.io/log-level: "info"                # Never "debug" in production
  dapr.io/config: "dapr-tracing-config"   # Tracing required for all services
  dapr.io/metrics-port: "9090"            # Standard metrics port
```

## Prohibited Patterns

Document what is explicitly not allowed:

```yaml
prohibited:
  - pattern: "Inline secrets in component metadata"
    example: |
      metadata:
        - name: password
          value: "mysecretpassword"  # NEVER
    correct: "Use secretKeyRef"

  - pattern: "Missing resiliency policy"
    rule: "All services calling external APIs must have a Resiliency resource"

  - pattern: "Wildcard scopes"
    rule: "All components must specify scopes (never allow all apps)"
```

## Standard Resiliency Policy

Define a base resiliency policy that all services inherit:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Resiliency
metadata:
  name: standard-resiliency
spec:
  policies:
    retries:
      standard:
        policy: exponential
        initialInterval: 500ms
        multiplier: 2
        maxInterval: 10s
        maxRetries: 3
    timeouts:
      standard: 30s
    circuitBreakers:
      standard:
        maxRequests: 5
        timeout: 30s
        trip: consecutiveFailures >= 5
  targets:
    apps:
      _default:           # Applied to all apps
        retry: standard
        timeout: standard
        circuitBreaker: standard
```

## Code Review Checklist

Add this to your PR template:

```markdown
## Dapr Checklist

- [ ] App ID follows naming convention (lowercase, hyphenated)
- [ ] No inline secrets in component YAML
- [ ] Scopes are defined on all components
- [ ] Resiliency policy is applied
- [ ] Tracing config annotation is present
- [ ] Component YAML is in the central templates repo, not duplicated
- [ ] Dead-letter topic configured for all pub/sub subscriptions
- [ ] initTimeout set on all components (default 30s is too long)
```

## Central Component Template Repository

Maintain a shared repository for component templates:

```bash
git clone git@github.com:yourorg/dapr-components.git

ls dapr-components/
# base/          - base resiliency and tracing configs
# state/         - state store templates per environment
# pubsub/        - pub/sub component templates
# secrets/       - secret store templates
# bindings/      - input/output binding templates
```

Teams pull from this repository rather than writing their own component YAML files.

## Summary

Dapr development standards covering naming conventions, mandatory annotations, prohibited patterns, and a standard resiliency policy create consistency across teams. A PR checklist and a central component template repository enforce standards at the code review stage and prevent configuration drift. Documenting what is explicitly not allowed (inline secrets, missing scopes) is as important as documenting what is required.
