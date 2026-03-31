# How to Use Dapr with Backstage Developer Portal

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Backstage, Developer Portal, Platform Engineering, Catalog

Description: Integrate Dapr with Backstage to create a developer portal that surfaces Dapr-enabled services, component templates, and operational dashboards in a unified interface.

---

## Backstage and Dapr Integration

Backstage is an open platform for building internal developer portals. Integrating Dapr with Backstage creates a central hub where developers can discover Dapr components, deploy new Dapr-enabled services from templates, and view Dapr health dashboards - all without leaving the portal.

## Installing Backstage

```bash
# Create a new Backstage app
npx @backstage/create-app@latest --skip-install
cd my-backstage-app
yarn install
yarn dev
```

## Adding Dapr-Enabled Services to the Catalog

Create catalog entities for Dapr-enabled services using the `catalog-info.yaml` convention:

```yaml
# order-service/catalog-info.yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: order-service
  description: "Order processing service with Dapr"
  annotations:
    dapr.io/app-id: "order-service"
    dapr.io/components: "redis-statestore,kafka-pubsub"
    backstage.io/kubernetes-label-selector: "app=order-service"
    github.com/project-slug: "myorg/order-service"
  tags:
    - dapr
    - golang
    - production
spec:
  type: service
  lifecycle: production
  owner: team-platform
  dependsOn:
    - component:redis-statestore
    - component:kafka-pubsub
```

Register Dapr components as catalog entities too:

```yaml
# catalog/dapr-components.yaml
apiVersion: backstage.io/v1alpha1
kind: Resource
metadata:
  name: redis-statestore
  description: "Shared Redis state store"
  annotations:
    dapr.io/component-type: "state.redis"
    dapr.io/certification: "stable"
spec:
  type: dapr-component
  owner: team-platform
  lifecycle: production
```

## Backstage Software Templates for Dapr

Create a Backstage template that scaffolds a new Dapr-enabled service:

```yaml
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: dapr-go-service
  title: "New Dapr Go Service"
  description: "Creates a new Go service with Dapr integration"
  tags: [dapr, golang]
spec:
  type: service
  parameters:
    - title: Service Configuration
      required: [serviceName, daprComponents]
      properties:
        serviceName:
          type: string
          title: Service Name
          pattern: "^[a-z][a-z0-9-]+$"
        daprComponents:
          type: array
          title: Required Dapr Components
          items:
            enum: [state, pubsub, bindings, secrets]
  steps:
    - id: fetch-template
      name: Fetch Service Template
      action: fetch:template
      input:
        url: ./skeleton
        values:
          serviceName: ${{ parameters.serviceName }}
          components: ${{ parameters.daprComponents }}
    - id: create-repo
      name: Create GitHub Repository
      action: github:repo:create
      input:
        repoUrl: github.com?owner=myorg&repo=${{ parameters.serviceName }}
    - id: register
      name: Register in Catalog
      action: catalog:register
      input:
        catalogInfoUrl: ${{ steps.create-repo.output.remoteUrl }}/blob/main/catalog-info.yaml
```

## Kubernetes Plugin for Dapr Status

Configure the Kubernetes plugin to show Dapr sidecar status:

```yaml
# app-config.yaml
kubernetes:
  serviceLocatorMethod:
    type: multiTenant
  clusterLocatorMethods:
    - type: config
      clusters:
        - url: ${K8S_CLUSTER_URL}
          name: production
          authProvider: serviceAccount
          serviceAccountToken: ${K8S_SA_TOKEN}
          customResources:
            - group: "dapr.io"
              apiVersion: "v1alpha1"
              plural: "components"
```

The Kubernetes tab in each service's Backstage page will show Dapr component status alongside pod health.

## TechDocs for Dapr Standards

Use Backstage TechDocs to publish your internal Dapr documentation:

```yaml
# dapr-docs/catalog-info.yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: dapr-standards
  annotations:
    backstage.io/techdocs-ref: dir:.
spec:
  type: documentation
  owner: team-platform
```

```bash
# Build and publish TechDocs
npx @techdocs/cli build
npx @techdocs/cli publish --publisher-type awsS3 --storage-name my-techdocs-bucket
```

## Grafana Dashboard Links

Add Dapr Grafana dashboard links to each service's Backstage page:

```yaml
metadata:
  annotations:
    grafana/dashboard-selector: "dapr_app_id='order-service'"
    grafana/overview-dashboard: "https://grafana.example.com/d/dapr-service/dapr-service?var-app_id=order-service"
```

## Summary

Integrating Dapr with Backstage creates a developer portal where engineers discover Dapr components as catalog resources, deploy new services from Dapr templates with scaffolded component YAML, and monitor Dapr sidecar health and metrics from the Kubernetes plugin. Publishing internal Dapr standards as TechDocs and linking Grafana dashboards per service reduces context switching and accelerates Dapr adoption across the organization.
