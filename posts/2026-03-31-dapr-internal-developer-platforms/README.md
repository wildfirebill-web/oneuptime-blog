# How to Use Dapr with Internal Developer Platforms

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Internal Developer Platform, IDP, Platform Engineering, Self-Service

Description: Integrate Dapr into your Internal Developer Platform to provide self-service Dapr component provisioning, standardized service templates, and automated environment setup.

---

## What Is an Internal Developer Platform?

An Internal Developer Platform (IDP) is a self-service layer that abstracts the complexity of infrastructure, enabling developers to deploy services, provision resources, and manage environments without waiting for platform team tickets. Dapr fits into an IDP as a managed capability.

## IDP Architecture with Dapr

```yaml
idp_layers:
  developer_interface:
    - Backstage portal (service catalog, templates)
    - CLI tool (internal "platform" CLI)
    - Slack bot (deploy, provision commands)

  orchestration:
    - Argo CD (GitOps for component YAML)
    - Crossplane (cloud resource provisioning)
    - GitHub Actions (CI/CD pipelines)

  dapr_layer:
    - Dapr control plane (platform team managed)
    - Component catalog (approved components)
    - Resiliency policies (standard policies applied automatically)

  infrastructure:
    - Kubernetes clusters
    - Redis, Kafka, Vault
    - Cloud services (RDS, SQS, etc.)
```

## Building a Platform CLI for Dapr

Create an internal CLI that wraps Dapr operations for developers:

```bash
# Internal CLI usage
platform service create \
  --name order-processor \
  --language go \
  --components state,pubsub

# Under the hood, this:
# 1. Creates a GitHub repo from the Dapr service template
# 2. Generates component YAML scoped to the new service
# 3. Opens a PR to the GitOps repo
# 4. Registers the service in the Backstage catalog
```

Example CLI implementation:

```go
// cmd/platform/main.go
package main

import (
    "fmt"
    "os/exec"
    "github.com/spf13/cobra"
)

var createCmd = &cobra.Command{
    Use:   "create",
    Short: "Create a new Dapr-enabled service",
    RunE: func(cmd *cobra.Command, args []string) error {
        name, _ := cmd.Flags().GetString("name")
        components, _ := cmd.Flags().GetStringSlice("components")

        // Generate component YAML from templates
        for _, comp := range components {
            fmt.Printf("Generating %s component for %s...\n", comp, name)
            generateComponent(comp, name)
        }

        // Create GitOps PR
        return createGitOpsPR(name, components)
    },
}
```

## Crossplane for Self-Service Infrastructure

Use Crossplane compositions to provision both cloud resources and Dapr components together:

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: dapr-redis-service
spec:
  compositeTypeRef:
    apiVersion: platform.mycompany.com/v1alpha1
    kind: DaprService
  resources:
    - name: redis-cluster
      base:
        apiVersion: elasticache.aws.crossplane.io/v1beta1
        kind: ReplicationGroup
        spec:
          forProvider:
            region: us-east-1
            nodeType: cache.t3.medium
      patches:
        - fromFieldPath: "spec.serviceName"
          toFieldPath: "metadata.name"

    - name: dapr-statestore-component
      base:
        apiVersion: dapr.io/v1alpha1
        kind: Component
        spec:
          type: state.redis
          version: v1
      patches:
        - fromFieldPath: "spec.serviceName"
          toFieldPath: "spec.scopes[0]"
```

A developer claims this composite resource:

```yaml
apiVersion: platform.mycompany.com/v1alpha1
kind: DaprService
metadata:
  name: order-processor
spec:
  serviceName: order-processor
  components:
    - state
```

## Environment Lifecycle Management

IDPs manage environments as first-class resources. Include Dapr component provisioning in environment creation:

```bash
# Create ephemeral preview environment with Dapr
platform env create \
  --name pr-456 \
  --from-pr 456 \
  --ttl 24h

# This provisions:
# - A new namespace: pr-456
# - Dapr components pointing to shared dev infrastructure
# - The service from the PR image
# - Automatic cleanup after 24h
```

Implementation using namespace provisioning:

```bash
#!/bin/bash
NAMESPACE="pr-${PR_NUMBER}"
kubectl create namespace "$NAMESPACE"
kubectl label namespace "$NAMESPACE" environment=preview ttl=24h

# Apply standard Dapr configuration
kubectl apply -f platform/dapr/base/ -n "$NAMESPACE"

# Apply service-specific components scoped to this namespace
envsubst < platform/dapr/preview/statestore.yaml \
  | kubectl apply -n "$NAMESPACE" -f -
```

## Developer Experience Metrics

Track how the IDP improves Dapr adoption:

```bash
# Time from "platform service create" to first successful deployment
# Measured via CI/CD pipeline timestamps

# Number of support tickets for Dapr issues per month
# Before IDP: 15/month; After: 3/month

# Developer satisfaction with Dapr onboarding (quarterly survey)
# Before IDP: 5.5/10; After: 8.2/10
```

## Summary

Integrating Dapr into an Internal Developer Platform enables self-service creation of Dapr-enabled services with automatically provisioned components, standardized configurations, and GitOps-driven deployment. Platform CLIs wrap complex Dapr operations into simple commands, Crossplane compositions tie cloud resource provisioning to Dapr component creation, and ephemeral preview environments with Dapr support accelerate feature development and review cycles.
