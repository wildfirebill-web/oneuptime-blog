# How to Create an Internal Dapr Developer Portal

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Developer Portal, Backstage, Platform Engineering, Documentation

Description: Build an internal Dapr developer portal using Backstage or a simple static site to surface component catalogs, templates, runbooks, and service dependency maps for your team.

---

## What an Internal Dapr Portal Provides

An internal Dapr developer portal gives engineers a single place to discover:
- Available Dapr components and their connection details
- Approved component templates for self-service adoption
- Service dependency maps showing which services use which components
- Runbooks for common Dapr operations
- Status of the Dapr control plane

## Option 1: Backstage-Based Portal

Backstage integrates natively with Kubernetes and can autodiscover Dapr-annotated services:

```bash
# Install Backstage
npx @backstage/create-app@latest --skip-install
cd my-portal
yarn install
```

Add the Kubernetes plugin to discover Dapr-enabled services:

```yaml
# app-config.yaml
kubernetes:
  serviceLocatorMethod:
    type: multiTenant
  clusterLocatorMethods:
    - type: config
      clusters:
        - url: https://k8s-cluster.example.com
          name: production
          authProvider: serviceAccount
          serviceAccountToken: ${K8S_SA_TOKEN}
```

Create a Dapr component template in Backstage:

```yaml
# templates/dapr-service/template.yaml
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: dapr-service
  title: "New Dapr-Enabled Service"
spec:
  type: service
  parameters:
    - title: Service Details
      properties:
        serviceName:
          type: string
          description: "Name of the service (lowercase, hyphenated)"
        buildingBlocks:
          type: array
          items:
            enum: [state, pubsub, bindings, secrets]
  steps:
    - id: generate
      action: fetch:template
      input:
        url: ./skeleton
        values:
          serviceName: ${{ parameters.serviceName }}
```

## Option 2: Simple Static Documentation Site

For teams not ready for Backstage, a MkDocs site works well:

```bash
pip install mkdocs mkdocs-material
mkdocs new dapr-portal
cd dapr-portal
```

Structure the site:

```yaml
# mkdocs.yml
site_name: "Dapr at OurCompany"
nav:
  - Home: index.md
  - Getting Started:
      - Local Setup: getting-started/local.md
      - First Service: getting-started/first-service.md
  - Components:
      - State Stores: components/state-stores.md
      - Pub/Sub: components/pubsub.md
      - Secrets: components/secrets.md
  - Templates: templates/index.md
  - Runbooks:
      - Upgrade Dapr: runbooks/upgrade.md
      - Debug Sidecar: runbooks/debug.md
```

## Component Catalog Page

Each component should have a catalog entry:

```markdown
## Redis State Store

- **Name**: `redis-statestore`
- **Type**: `state.redis`
- **Environments**: dev, staging, production
- **Host**: `redis-master.production.svc:6379`
- **Certification Tier**: Stable
- **Approved Scopes**: Must be requested via platform-team ticket

### Usage

Apply the component template from the templates repo:

kubectl apply -k github.com/myorg/dapr-components//state/redis/overlays/production

### Runbook

If this component fails, check: [Redis State Store Runbook](../runbooks/redis-state.md)
```

## Service Dependency Map

Generate a dependency map using the Dapr metadata API:

```bash
#!/bin/bash
# Generate service-to-component dependency report

kubectl get pods --all-namespaces -l dapr.io/enabled=true \
  -o jsonpath='{range .items[*]}{.metadata.annotations.dapr\.io/app-id}{"\n"}{end}' \
  | sort -u | while read app_id; do
    echo "=== $app_id ==="
    kubectl exec deploy/$app_id -c daprd -- \
      wget -qO- http://localhost:3500/v1.0/metadata 2>/dev/null \
      | jq -r '.components[].name'
  done
```

## Keeping the Portal Updated

Automate portal updates using CI:

```yaml
# .github/workflows/update-portal.yaml
name: Update Dapr Portal
on:
  push:
    paths:
      - 'components/**'
jobs:
  update:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Rebuild component catalog
        run: python scripts/generate-catalog.py
      - name: Deploy to portal
        run: mkdocs gh-deploy
```

## Summary

An internal Dapr developer portal - whether Backstage-based or a simple MkDocs site - reduces friction by centralizing component discovery, self-service templates, runbooks, and service dependency maps. Starting with a simple static site and automating updates via CI is often faster than deploying Backstage and provides immediate value to application teams adopting Dapr.
