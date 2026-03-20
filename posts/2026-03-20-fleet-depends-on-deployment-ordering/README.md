# How to Configure Fleet Depends-On for Deployment Ordering

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Fleet, GitOps, Rancher, Kubernetes, Dependencies

Description: Learn how to use Fleet's dependsOn feature to control the deployment order of bundles, ensuring that dependent applications are only deployed after their prerequisites are ready.

## Introduction

In complex application environments, some components must be deployed before others. A database must exist before the application server, an ingress controller must be running before application ingresses, and CRDs must be applied before the operators that use them. Fleet's `dependsOn` feature lets you define these ordering requirements explicitly.

This guide covers how to configure bundle dependencies to ensure correct deployment ordering across your clusters.

## Prerequisites

- Fleet installed in Rancher
- Multiple related applications in your Git repository
- `kubectl` access to Fleet manager
- Understanding of Fleet bundle concepts

## Understanding dependsOn

The `dependsOn` field in `fleet.yaml` tells Fleet that a bundle should not be deployed until the specified bundles are in the `Ready` state. Fleet checks:
1. The referenced bundle exists
2. The referenced bundle has `status.summary.ready` equal to `status.summary.desiredReady`
3. No clusters are in `NotReady` or `Modified` state for the referenced bundle

## Basic Dependency Configuration

Consider this application stack:
1. **Namespace** - Must be created first
2. **Database** - Requires namespace, deployed second
3. **Backend API** - Requires database, deployed third
4. **Frontend** - Requires backend API, deployed last

```text
app-stack/
├── 01-namespaces/
│   ├── fleet.yaml
│   └── namespaces.yaml
├── 02-database/
│   ├── fleet.yaml          # dependsOn: namespaces
│   ├── statefulset.yaml
│   └── service.yaml
├── 03-backend/
│   ├── fleet.yaml          # dependsOn: database
│   ├── deployment.yaml
│   └── service.yaml
└── 04-frontend/
    ├── fleet.yaml          # dependsOn: backend
    ├── deployment.yaml
    └── service.yaml
```

### Configuring fleet.yaml with dependsOn

```yaml
# 01-namespaces/fleet.yaml

namespace: ""  # Namespaces don't need a target namespace
targets:
  - clusterSelector: {}
```

```yaml
# 02-database/fleet.yaml
namespace: my-app

# Wait for namespaces bundle to be Ready before deploying
dependsOn:
  - name: app-stack-01-namespaces
    namespace: fleet-default

targets:
  - clusterSelector: {}
```

```yaml
# 03-backend/fleet.yaml
namespace: my-app

# Wait for database bundle to be Ready
dependsOn:
  - name: app-stack-02-database
    namespace: fleet-default

targets:
  - clusterSelector: {}
```

```yaml
# 04-frontend/fleet.yaml
namespace: my-app

# Wait for backend to be Ready
dependsOn:
  - name: app-stack-03-backend
    namespace: fleet-default

targets:
  - clusterSelector: {}
```

## Multiple Dependencies

A bundle can depend on multiple other bundles. All must be ready:

```yaml
# full-app/fleet.yaml
namespace: my-app

# Deploy only when ALL of these bundles are ready
dependsOn:
  # Database must be ready
  - name: app-stack-02-database
    namespace: fleet-default
  # Message queue must be ready
  - name: platform-rabbitmq
    namespace: fleet-default
  # Shared config must be ready
  - name: platform-shared-config
    namespace: fleet-default

targets:
  - clusterSelector: {}
```

## Cross-GitRepo Dependencies

Dependencies can reference bundles from different GitRepo resources:

```yaml
# apps/my-service/fleet.yaml
namespace: my-service

# Depend on infrastructure bundles from a different GitRepo
dependsOn:
  # Wait for the ingress controller (from infrastructure GitRepo)
  - name: infrastructure-ingress-nginx
    namespace: fleet-default

  # Wait for cert-manager (from platform GitRepo)
  - name: platform-cert-manager
    namespace: fleet-default

targets:
  - clusterSelector:
      matchLabels:
        env: production
```

## Practical Example: Installing a Full Platform Stack

For a complete platform with ordered components:

```text
platform/
├── crds/
│   ├── fleet.yaml
│   └── custom-crds.yaml
├── cert-manager/
│   ├── fleet.yaml          # dependsOn: crds
│   └── cert-manager.yaml
├── ingress-nginx/
│   ├── fleet.yaml          # dependsOn: crds
│   └── ingress-nginx.yaml
├── monitoring/
│   ├── fleet.yaml          # dependsOn: crds, ingress-nginx
│   └── prometheus-stack.yaml
└── applications/
    ├── fleet.yaml          # dependsOn: cert-manager, ingress-nginx, monitoring
    └── app-manifests.yaml
```

```yaml
# monitoring/fleet.yaml
namespace: monitoring

dependsOn:
  # Monitoring requires CRDs to be installed first
  - name: platform-crds
    namespace: fleet-default
  # Monitoring ingress requires ingress controller
  - name: platform-ingress-nginx
    namespace: fleet-default

targets:
  - clusterSelector: {}
```

```yaml
# applications/fleet.yaml
namespace: applications

dependsOn:
  # All platform components must be ready first
  - name: platform-cert-manager
    namespace: fleet-default
  - name: platform-ingress-nginx
    namespace: fleet-default
  - name: platform-monitoring
    namespace: fleet-default

targets:
  - clusterSelector: {}
```

## Viewing Dependency Relationships

```bash
# Check if a bundle is waiting on dependencies
kubectl describe bundle app-stack-03-backend -n fleet-default \
  | grep -A 10 "Waiting"

# View the dependsOn configuration of a bundle
kubectl get bundle app-stack-03-backend -n fleet-default \
  -o jsonpath='{.spec.dependsOn}'

# Check dependency status
kubectl get bundles -n fleet-default \
  -o jsonpath='{range .items[*]}{.metadata.name}: ready={.status.summary.ready}/{.status.summary.desiredReady}{"\n"}{end}'
```

## Troubleshooting Dependency Issues

```bash
# Bundle stuck waiting? Check the dependency
kubectl describe bundle app-stack-03-backend -n fleet-default

# Verify the dependency bundle exists and is ready
kubectl get bundle app-stack-02-database -n fleet-default

# Check for naming mismatches
# Bundle names follow the pattern: <gitrepo-name>-<path-with-dashes>
kubectl get bundles -n fleet-default | grep "app-stack"

# Force a re-evaluation
kubectl annotate gitrepo my-app-stack \
  -n fleet-default \
  fleet.cattle.io/commit="" \
  --overwrite
```

## Conclusion

Fleet's `dependsOn` feature enables reliable ordered deployments across complex application stacks. By explicitly declaring dependencies between bundles, you prevent race conditions where application components start before their prerequisites are ready. Whether you're deploying a simple three-tier application or a complex platform with dozens of interdependent components, `dependsOn` gives you the control needed to ensure everything deploys in the right order across all your clusters simultaneously.
