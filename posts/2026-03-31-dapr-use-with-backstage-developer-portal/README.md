# How to Use Dapr with Backstage Developer Portal

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Backstage, Developer Portal, Catalog, Platform Engineering

Description: Learn how to integrate Dapr into Backstage by registering Dapr components and services in the software catalog and creating scaffolding templates for new Dapr-enabled services.

---

Backstage is a CNCF-graduated developer portal framework. Integrating Dapr into Backstage lets your teams discover available components, understand service dependencies, and scaffold new Dapr-enabled services through a self-service UI.

## Backstage Software Catalog for Dapr

Register Dapr infrastructure components as Backstage catalog entities so teams can discover what is available:

```yaml
# catalog/dapr-statestore-prod.yaml
apiVersion: backstage.io/v1alpha1
kind: Resource
metadata:
  name: statestore-prod
  title: "Production State Store (Redis)"
  description: "Redis-backed Dapr state store for production services"
  annotations:
    backstage.io/managed-by-location: url:https://github.com/myorg/gitops/blob/main/components/statestore-prod.yaml
  tags:
    - dapr
    - state-store
    - redis
    - production
  links:
    - url: https://docs.dapr.io/reference/components-reference/supported-state-stores/
      title: Dapr State Store Docs
spec:
  type: dapr-component
  lifecycle: production
  owner: platform-team
```

```yaml
# catalog/order-service.yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: order-service
  description: "Processes customer orders using Dapr pub/sub and state management"
  annotations:
    dapr.io/app-id: "payments-order-service"
  tags:
    - dapr
    - payments
spec:
  type: service
  lifecycle: production
  owner: payments-team
  consumesApis: []
  dependsOn:
    - resource:statestore-prod
    - resource:pubsub-prod
```

## Registering the Catalog

Add the catalog files to your `app-config.yaml`:

```yaml
catalog:
  locations:
    - type: url
      target: https://github.com/myorg/gitops/blob/main/catalog/all-components.yaml
    - type: file
      target: ./catalog/dapr-components.yaml
```

## Backstage Scaffolding Template for New Dapr Services

Create a software template that scaffolds a complete Dapr-enabled service:

```yaml
# templates/dapr-service-template.yaml
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: dapr-service
  title: "New Dapr-Enabled Service"
  description: "Create a new Go/Python/Node.js service with Dapr sidecar pre-configured"
  tags:
    - dapr
    - recommended
spec:
  owner: platform-team
  type: service
  parameters:
    - title: Service Details
      required: [serviceName, teamName, language]
      properties:
        serviceName:
          type: string
          title: Service Name
          pattern: '^[a-z][a-z0-9-]*$'
        teamName:
          type: string
          title: Team Name
        language:
          type: string
          title: Language
          enum: [go, python, nodejs]
  steps:
    - id: fetch-template
      name: Fetch Service Template
      action: fetch:template
      input:
        url: ./skeleton/${{ parameters.language }}
        values:
          serviceName: ${{ parameters.serviceName }}
          teamName: ${{ parameters.teamName }}
          appId: "${{ parameters.teamName }}-${{ parameters.serviceName }}"

    - id: publish
      name: Publish to GitHub
      action: publish:github
      input:
        repoUrl: github.com?owner=myorg&repo=${{ parameters.serviceName }}
        defaultBranch: main

    - id: register
      name: Register in Catalog
      action: catalog:register
      input:
        repoContentsUrl: ${{ steps.publish.output.repoContentsUrl }}
        catalogInfoPath: /catalog-info.yaml
```

## Viewing Dapr Dependencies in Backstage

Once catalog entities reference Dapr components via `dependsOn`, the Backstage dependency graph shows which services use which Dapr components:

```
order-service
  depends on: statestore-prod (Redis)
  depends on: pubsub-prod (Kafka)
  depends on: secrets-prod (Vault)
```

This helps the platform team understand the blast radius of component changes.

## TechDocs for Dapr Runbooks

Add runbooks as Backstage TechDocs:

```yaml
metadata:
  annotations:
    backstage.io/techdocs-ref: dir:.
```

Place your Dapr troubleshooting docs in `docs/` and they appear in Backstage's documentation tab.

## Summary

Integrating Dapr with Backstage involves registering Dapr components and services as catalog entities, using the `dependsOn` relationship to model which services use which Dapr components, and creating scaffolding templates that generate new Dapr-enabled services through the Backstage UI. This reduces onboarding time and centralizes discovery for the entire engineering organization.
