# How to Inspect Helm Chart Details in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Helm, DevOps

Description: Learn how to inspect Helm chart details, view release history, and examine deployed resources using Portainer's Helm interface.

## Introduction

After deploying Helm charts in Portainer, you need visibility into what was deployed, what values were used, and the history of changes. Portainer provides views for inspecting Helm releases and their associated Kubernetes resources. This guide covers navigating chart details in Portainer.

## Prerequisites

- Portainer with Kubernetes environment
- At least one Helm chart deployed

## Step 1: View Helm Release List

1. Select your Kubernetes environment in Portainer
2. Navigate to **Applications → Helm** or look for Helm releases in the Applications section

The release list shows:

```
RELEASE         NAMESPACE     CHART                    VERSION   STATUS     AGE
nginx-ingress   ingress-nginx nginx-ingress-controller  4.8.0    deployed   3d
prometheus      monitoring    kube-prometheus-stack     54.0.0   deployed   7d
cert-manager    cert-manager  cert-manager              v1.13.2  deployed   30d
my-app          production    my-app-chart              1.2.0    deployed   1d
```

## Step 2: Inspect a Helm Release

Click on a release name to see detailed information:

```
Release: nginx-ingress
──────────────────────────────────────────────────
Chart:          nginx-ingress-controller-4.8.0
Status:         deployed
Revision:       3
Last deployed:  2024-01-15 10:00:00
Namespace:      ingress-nginx
App version:    1.9.4
```

## Step 3: View Release Notes

Helm chart notes appear after deployment and provide important post-install instructions:

```
NOTES:
The nginx-ingress-controller has been installed.

Get the application URL by running these commands:
  export SERVICE_IP=$(kubectl get svc nginx-ingress-nginx-ingress-controller \
    -n ingress-nginx -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
  echo "Visit http://$SERVICE_IP to access your application"

WARNING: Kubernetes v1.18+ uses `networking.k8s.io/v1` for Ingress resources.
```

## Step 4: View Current Values

See the values used to deploy the chart:

In Portainer, click **Values** or **User supplied values** tab:

```yaml
replicaCount: 2
service:
  type: LoadBalancer
controller:
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
```

```bash
# CLI equivalent
helm get values nginx-ingress --namespace ingress-nginx

# Show ALL values (user + defaults)
helm get values nginx-ingress --namespace ingress-nginx --all
```

## Step 5: View Chart Default Values

See all configurable values with their defaults:

```bash
# Show all chart values (not release-specific)
helm show values bitnami/nginx-ingress-controller

# Output excerpt:
# ## @section Global parameters
# global:
#   imageRegistry: ""
#   imagePullSecrets: []
#   storageClass: ""
#
# replicaCount: 1
# ...
```

## Step 6: View Generated Manifests

See the Kubernetes manifests Helm generated for this release:

```bash
# View all manifests in the release
helm get manifest nginx-ingress --namespace ingress-nginx

# Output: all the YAML that was applied to the cluster
```

In Portainer, click on individual resources from the release detail to see their YAML.

## Step 7: View Release History

See all revisions of the release:

```bash
helm history nginx-ingress --namespace ingress-nginx

# Output:
# REVISION  UPDATED                  STATUS     CHART                      DESCRIPTION
# 1         2024-01-10 09:00:00      superseded nginx-ingress-c-4.7.0      Install complete
# 2         2024-01-12 14:00:00      superseded nginx-ingress-c-4.7.0      Upgrade complete
# 3         2024-01-15 10:00:00      deployed   nginx-ingress-c-4.8.0      Upgrade complete
```

## Step 8: Compare Values Between Revisions

```bash
# Get values from a specific revision
helm get values nginx-ingress --revision 2 --namespace ingress-nginx

# Diff between revisions (use helm diff plugin)
helm diff revision nginx-ingress 2 3 --namespace ingress-nginx
```

## Step 9: View Kubernetes Resources Created by the Chart

In Portainer, navigate to the namespace and filter by Helm release labels:

```bash
# CLI: See all resources in a release
helm get manifest nginx-ingress -n ingress-nginx | grep "^kind:"

# Or use kubectl with Helm labels
kubectl get all -n ingress-nginx \
  -l "app.kubernetes.io/instance=nginx-ingress"
```

## Step 10: Inspect Chart README and Schema

```bash
# View chart README
helm show readme bitnami/nginx-ingress-controller

# View chart values schema (if provided)
helm show chart bitnami/nginx-ingress-controller

# Get chart metadata
helm inspect chart bitnami/nginx-ingress-controller
# Output:
# apiVersion: v2
# appVersion: 1.9.4
# description: NGINX Ingress Controller...
# version: 4.8.0
```

## Step 11: Check for Chart Updates

```bash
# Update repository index
helm repo update

# Search for newer versions
helm search repo bitnami/nginx-ingress-controller --versions | head -5

# Check if installed chart has updates available
helm list --namespace ingress-nginx | grep nginx-ingress
# Compare CHART column with latest from search
```

In Portainer, check the chart version in the release details and compare with the latest in the repository.

## Conclusion

Portainer provides accessible views of Helm release details that would otherwise require multiple kubectl and helm commands. Use the release detail view to understand what's deployed, check values to understand configuration, and monitor release history for audit purposes. For complex chart inspection needs (schema, README, value schema), the Helm CLI remains the most comprehensive tool.
