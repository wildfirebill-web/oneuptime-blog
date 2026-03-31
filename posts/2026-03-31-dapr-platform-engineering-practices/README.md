# How to Use Dapr with Platform Engineering Practices

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Platform Engineering, Golden Path, Self-Service, Developer Experience

Description: Apply platform engineering principles to Dapr adoption by building golden paths, self-service templates, paved roads, and automated guardrails that accelerate developer productivity.

---

## Dapr as a Platform Capability

Platform engineering focuses on creating reusable, self-service infrastructure capabilities that reduce cognitive load for application developers. Dapr fits naturally as a platform capability - the platform team owns Dapr operations while application teams consume building blocks through a standardized interface.

## The Golden Path for New Dapr Services

A golden path is a paved, opinionated route that guides developers to the right patterns without restricting them. For Dapr, the golden path includes:

```yaml
golden_path:
  step_1: "Use the service scaffold template from the internal portal"
  step_2: "Select required Dapr components from the approved catalog"
  step_3: "Component YAML is auto-generated from the template"
  step_4: "CI pipeline validates component configuration"
  step_5: "GitOps deploys components before the service"
  step_6: "Observability is auto-configured via the standard Dapr config"
```

## Service Scaffold Template

Create a Cookiecutter or Backstage template that generates a ready-to-run Dapr service:

```bash
# Install cookiecutter
pip install cookiecutter

# Use internal Dapr service template
cookiecutter git@github.com:myorg/dapr-service-template.git

# Prompts:
# service_name [my-service]: order-processor
# language [go/python/java]: go
# components [state,pubsub,secrets]: state,pubsub
```

Generated structure:

```text
order-processor/
├── cmd/main.go               # Service entrypoint with Dapr client
├── components/
│   ├── statestore.yaml       # Pre-filled from organization standards
│   └── pubsub.yaml
├── config/
│   └── dapr-config.yaml      # Standard tracing + resiliency
├── Dockerfile                # Multi-arch build
├── catalog-info.yaml         # Backstage registration
└── score.yaml                # Score workload spec
```

## Automated Guardrails with OPA

Use Open Policy Agent (OPA) to enforce Dapr standards at admission time:

```rego
# rego/dapr-standards.rego
package dapr.standards

deny[msg] {
    input.kind == "Component"
    not input.spec.metadata[_].name == "redisPassword"
    not input.spec.auth.secretStore
    msg := "Dapr component must use auth.secretStore for secret resolution"
}

deny[msg] {
    input.kind == "Component"
    count(input.spec.scopes) == 0
    msg := "Dapr component must define at least one scope"
}

deny[msg] {
    input.kind == "Component"
    not input.spec.initTimeout
    msg := "Dapr component must set initTimeout"
}
```

Deploy with Gatekeeper:

```bash
kubectl apply -f opa/dapr-standards-constraint.yaml
kubectl apply -f opa/dapr-standards-template.yaml
```

Any component YAML that violates the policy will be rejected at `kubectl apply` time.

## Self-Service Component Provisioning

Build a self-service workflow for requesting new Dapr components:

```yaml
# GitHub Issue template: .github/ISSUE_TEMPLATE/dapr-component-request.yaml
name: Dapr Component Request
description: Request a new Dapr component for your service
body:
  - type: input
    id: service_name
    label: Service Name (Dapr app-id)
  - type: dropdown
    id: component_type
    label: Component Type
    options: [state.redis, pubsub.kafka, secretstores.vault, bindings.s3]
  - type: input
    id: justification
    label: Business Justification
```

A GitHub Actions workflow processes the request, creates the component YAML via a PR, and notifies the requestor.

## Measuring Platform Effectiveness

Track the health of your Dapr platform via metrics:

```bash
#!/bin/bash
echo "=== Dapr Platform Health Report ==="

# Services onboarded
echo "Dapr-enabled services:"
kubectl get pods --all-namespaces -l dapr.io/enabled=true --no-headers | wc -l

# Component adoption
echo "Components in use:"
kubectl get components --all-namespaces --no-headers | wc -l

# Policy violations caught this month
echo "OPA policy violations:"
kubectl get events --all-namespaces \
  --field-selector reason=FailedCreate \
  | grep dapr | wc -l
```

## Documentation as a Platform Product

Treat internal Dapr documentation as a product with user feedback:

```yaml
documentation_standards:
  format: "MkDocs with Material theme"
  review_cycle: "Quarterly or after each Dapr upgrade"
  feedback: "Thumbs up/down on each page with comment box"
  owners: "Platform team with quarterly review of feedback"
  sla: "New component documented within 5 business days of approval"
```

## Summary

Applying platform engineering practices to Dapr means building golden paths with service scaffold templates, enforcing standards automatically with OPA admission policies, enabling self-service component provisioning via issue workflows, and measuring platform effectiveness through adoption and policy metrics. The goal is to make the right Dapr patterns effortless for application developers while giving the platform team operational control and visibility.
