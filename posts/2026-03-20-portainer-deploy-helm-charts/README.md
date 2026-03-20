# How to Deploy Helm Charts in Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Helm, Deployment, DevOps

Description: Learn how to browse, configure, and deploy Helm charts to Kubernetes clusters using Portainer's built-in Helm support.

## Introduction

Portainer integrates with Helm to provide a graphical interface for deploying Helm charts to Kubernetes clusters. Instead of running `helm install` commands with complex value flags, you can browse chart repositories, configure values in a form, and deploy with a click. This guide covers the complete Helm deployment workflow in Portainer.

## Prerequisites

- Portainer CE or BE with Kubernetes environment connected
- Cluster accessible from Portainer
- Helm chart repositories configured (or using built-in ones)

## Step 1: Navigate to Helm in Portainer

1. Select your Kubernetes environment
2. Click **Applications → Helm charts**
3. The Helm chart catalog appears

## Step 2: Add Helm Repositories

Portainer comes with some built-in Helm repositories. Add more:

1. Go to **Settings → Helm repositories** (or within the Kubernetes environment)
2. Click **Add repository**
3. Enter repository details:

```text
Name:   bitnami
URL:    https://charts.bitnami.com/bitnami
```

Popular repositories to add:

```bash
# Add common Helm repositories

helm repo add stable https://charts.helm.sh/stable
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add cert-manager https://charts.jetstack.io
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
```

## Step 3: Search for a Chart

In the Portainer Helm catalog:

1. Use the search bar to find charts
2. Filter by repository
3. Click on a chart to see description and available versions

Example: Search for "nginx-ingress":

```text
nginx-ingress-controller    bitnami     4.8.0    NGINX Ingress Controller
ingress-nginx               ingress-nginx  4.8.4  Ingress controller for Kubernetes
```

## Step 4: Deploy a Helm Chart

Click on a chart, then click **Install**:

```text
Chart:       nginx-ingress-controller
Version:     4.8.0
Repository:  bitnami
Namespace:   ingress-nginx    (create if needed: [x])
Release name: nginx-ingress
```

## Step 5: Configure Chart Values

The **Values** section shows customizable options:

### Method A: Form-Based Values

Some charts show a form with labeled fields. Fill in the values:

```text
Replica count:         2
Service type:          LoadBalancer
Enable HTTPS:          [x]
```

### Method B: YAML Values Editor

For full control, click **Custom values (YAML)**:

```yaml
# Custom values.yaml
replicaCount: 2

service:
  type: LoadBalancer
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: nlb

controller:
  config:
    proxy-body-size: "50m"
    proxy-read-timeout: "3600"
    proxy-send-timeout: "3600"
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 1000m
      memory: 512Mi
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 10
    targetCPUUtilizationPercentage: 80
```

## Step 6: Preview the Manifest (Optional)

Before deploying, preview what Helm will create:

```bash
# CLI equivalent: dry run
helm install nginx-ingress bitnami/nginx-ingress-controller \
  --namespace ingress-nginx \
  --values custom-values.yaml \
  --dry-run

# Template rendering
helm template nginx-ingress bitnami/nginx-ingress-controller \
  --values custom-values.yaml
```

## Step 7: Install the Chart

1. Review settings
2. Click **Install**
3. Watch the deployment output

Portainer shows the resources being created:

```text
NAME: nginx-ingress
LAST DEPLOYED: Tue Jan 15 2024 10:00:00
NAMESPACE: ingress-nginx
STATUS: deployed
REVISION: 1

Resources created:
  Deployment/nginx-ingress-nginx-ingress-controller
  Service/nginx-ingress-nginx-ingress-controller
  ServiceAccount/nginx-ingress-nginx-ingress-controller
  ClusterRole/nginx-ingress-nginx-ingress-controller
  ClusterRoleBinding/nginx-ingress-nginx-ingress-controller
```

## Step 8: View Deployed Helm Releases

Go to **Applications → Helm releases** (or check the Helm section):

```text
RELEASE NAME     NAMESPACE      CHART                          VERSION   STATUS
nginx-ingress    ingress-nginx  nginx-ingress-controller       4.8.0     deployed
cert-manager     cert-manager   cert-manager                   v1.13.2   deployed
prometheus       monitoring     kube-prometheus-stack           54.2.2    deployed
```

## Step 9: Upgrade a Helm Release

1. Click on the Helm release
2. Click **Upgrade**
3. Change the chart version and/or values
4. Click **Upgrade**

```bash
# CLI equivalent
helm upgrade nginx-ingress bitnami/nginx-ingress-controller \
  --namespace ingress-nginx \
  --version 4.9.0 \
  --values custom-values.yaml
```

## Step 10: Rollback a Helm Release

If an upgrade fails:

```bash
# Rollback to previous version
helm rollback nginx-ingress --namespace ingress-nginx

# Rollback to specific revision
helm rollback nginx-ingress 1 --namespace ingress-nginx
```

In Portainer: click on the Helm release and use the **Rollback** option.

## Step 11: Uninstall a Helm Release

1. Click on the Helm release in Portainer
2. Click **Uninstall** or **Remove**
3. Confirm

```bash
# CLI equivalent
helm uninstall nginx-ingress --namespace ingress-nginx
```

## Conclusion

Portainer's Helm integration brings the power of the Kubernetes package ecosystem to a graphical interface. Browse repositories, configure values visually, and deploy charts without remembering complex CLI syntax. For teams adopting Kubernetes, Helm through Portainer provides an excellent on-ramp that removes the command-line barrier while delivering all the benefits of the Helm ecosystem.
