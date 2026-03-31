# How to Deploy Dapr on Google Kubernetes Engine (GKE)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, GKE, Kubernetes, GCP, Deployment

Description: Step-by-step guide to deploying Dapr on Google Kubernetes Engine with Workload Identity, HA mode, and production-ready configuration.

---

## Overview

Google Kubernetes Engine (GKE) is a fully managed Kubernetes service on GCP. Deploying Dapr on GKE enables your microservices to leverage Dapr's building blocks while benefiting from GCP's managed infrastructure. This guide covers everything from cluster setup to production-ready Dapr deployment.

## Prerequisites

- `gcloud` CLI installed and authenticated
- `kubectl` installed
- `helm` v3 installed
- Dapr CLI installed

## Step 1: Create a GKE Cluster

```bash
gcloud container clusters create dapr-cluster \
  --region=us-central1 \
  --num-nodes=3 \
  --machine-type=e2-standard-4 \
  --workload-pool=my-project.svc.id.goog \
  --enable-autoscaling \
  --min-nodes=2 \
  --max-nodes=6
```

Get cluster credentials:

```bash
gcloud container clusters get-credentials dapr-cluster \
  --region=us-central1
```

## Step 2: Install Dapr with Helm

Add the Dapr Helm repository:

```bash
helm repo add dapr https://dapr.github.io/helm-charts/
helm repo update
```

Install Dapr in high-availability mode:

```bash
helm upgrade --install dapr dapr/dapr \
  --version=1.13.0 \
  --namespace dapr-system \
  --create-namespace \
  --set global.ha.enabled=true \
  --set dapr_operator.logLevel=info \
  --set dapr_sentry.logLevel=info \
  --wait
```

Verify the installation:

```bash
kubectl get pods -n dapr-system
```

Expected output:

```
NAME                                     READY   STATUS    RESTARTS
dapr-dashboard-5d7b47f6-4xjks           1/1     Running   0
dapr-operator-7d6b9d8b4-pqrst           1/1     Running   0
dapr-placement-server-0                 1/1     Running   0
dapr-sentry-6c9f7d8b4-mnopq             1/1     Running   0
dapr-sidecar-injector-7f9b8d4c-uvwxy   1/1     Running   0
```

## Step 3: Configure mTLS

Dapr uses mTLS for inter-service communication by default. Verify it's enabled:

```bash
kubectl get configuration -n default -o yaml
```

To customize the mTLS certificate rotation:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: appconfig
spec:
  mtls:
    enabled: true
    workloadCertTTL: "24h"
    allowedClockSkew: "15m"
```

## Step 4: Deploy a Sample Application

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-dapr
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello-dapr
  template:
    metadata:
      labels:
        app: hello-dapr
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "hello-dapr"
        dapr.io/app-port: "8080"
        dapr.io/log-level: "info"
    spec:
      containers:
      - name: hello-dapr
        image: gcr.io/my-project/hello-dapr:latest
        ports:
        - containerPort: 8080
```

## Step 5: Enable Dapr Dashboard

```bash
kubectl port-forward svc/dapr-dashboard -n dapr-system 8080:8080
```

Access the dashboard at `http://localhost:8080`.

## Summary

Deploying Dapr on GKE requires creating a cluster with Workload Identity, installing Dapr via Helm in HA mode, and annotating your deployments. With GKE's managed infrastructure and Dapr's building blocks, you get a production-ready microservices platform that scales automatically.
