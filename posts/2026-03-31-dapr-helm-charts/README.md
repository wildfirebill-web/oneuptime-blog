# How to Use Dapr with Helm Charts

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Helm, Kubernetes, Deployment, Package Management

Description: Package and deploy Dapr microservices and component configurations using Helm charts for repeatable, versioned Kubernetes deployments.

---

## Overview

Helm is the package manager for Kubernetes. Combining Helm with Dapr allows teams to package microservice deployments complete with Dapr annotations, component configurations, and subscriptions into reusable, versioned charts. This guide covers both installing Dapr via Helm and packaging your own Dapr applications as Helm charts.

## Prerequisites

- Helm v3 installed
- Kubernetes cluster running
- Dapr Helm repository added

## Installing Dapr via Helm

```bash
# Add the Dapr Helm repository
helm repo add dapr https://dapr.github.io/helm-charts/
helm repo update

# View available versions
helm search repo dapr --versions | head -10

# Install Dapr with custom values
helm upgrade --install dapr dapr/dapr \
  --version 1.13.0 \
  --namespace dapr-system \
  --create-namespace \
  --values dapr-values.yaml \
  --wait
```

Custom values file:

```yaml
global:
  ha:
    enabled: true
  logAsJson: true
  logLevel: info

dapr_operator:
  replicaCount: 3

dapr_sentry:
  replicaCount: 3
```

## Creating a Helm Chart for a Dapr Application

```bash
helm create order-service-chart
cd order-service-chart
```

Update `values.yaml`:

```yaml
image:
  repository: myorg/order-service
  tag: "1.0.0"
  pullPolicy: IfNotPresent

dapr:
  enabled: true
  appId: order-service
  appPort: 8080
  logLevel: info
  components:
    statestore: statestore
    pubsub: pubsub

replicaCount: 2
```

## Deployment Template with Dapr Annotations

`templates/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "order-service.fullname" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "order-service.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "order-service.selectorLabels" . | nindent 8 }}
      {{- if .Values.dapr.enabled }}
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "{{ .Values.dapr.appId }}"
        dapr.io/app-port: "{{ .Values.dapr.appPort }}"
        dapr.io/log-level: "{{ .Values.dapr.logLevel }}"
        dapr.io/log-as-json: "true"
      {{- end }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        ports:
        - containerPort: {{ .Values.dapr.appPort }}
```

## Dapr Component as a Helm Template

`templates/dapr-statestore.yaml`:

```yaml
{{- if .Values.dapr.enabled }}
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: {{ .Values.dapr.components.statestore }}
  namespace: {{ .Release.Namespace }}
spec:
  type: state.redis
  version: v1
  metadata:
  - name: redisHost
    value: "{{ .Values.redis.host }}:6379"
  - name: redisPassword
    secretKeyRef:
      name: {{ .Release.Name }}-redis-secret
      key: password
{{- end }}
```

## Packaging and Deploying

```bash
# Package the chart
helm package order-service-chart --version 1.0.0

# Install with environment-specific values
helm upgrade --install order-service ./order-service-chart \
  --namespace production \
  --values values-production.yaml \
  --set image.tag=v1.2.3

# Verify deployment
helm status order-service -n production
```

## Summary

Helm charts provide a clean packaging mechanism for Dapr microservices, bundling Dapr annotations, component configurations, and application deployments into versioned, reusable packages. Using Helm's templating for Dapr annotations enables consistent deployments across environments with environment-specific value overrides.
