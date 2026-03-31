# How to Set Up a Dapr Platform Team

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Platform Engineering, Team, Operations, Kubernetes

Description: Learn how to structure and operate a Dapr platform team responsible for installing, upgrading, securing, and supporting Dapr across multiple application teams.

---

## What Is a Dapr Platform Team?

A Dapr platform team is an internal group responsible for the Dapr control plane and shared components. They own the infrastructure so application teams can focus on building features instead of managing Dapr runtime operations.

## Responsibilities

```yaml
platform_team_responsibilities:
  install_and_upgrade:
    - "Deploy Dapr control plane with Helm"
    - "Plan and execute minor version upgrades"
    - "Maintain Dapr Helm values per cluster"

  component_management:
    - "Publish approved component templates"
    - "Review and approve new component requests"
    - "Enforce component certification tier standards"

  security:
    - "Configure mTLS between sidecars"
    - "Manage secret store access policies"
    - "Review and apply Dapr RBAC"

  observability:
    - "Maintain Prometheus scrape configs for Dapr metrics"
    - "Manage Grafana dashboards for Dapr health"
    - "Set up alerting for control plane health"

  developer_support:
    - "Run internal Dapr office hours"
    - "Maintain internal docs and component template repo"
    - "Triage Dapr-related support tickets"
```

## Installing Dapr with Production Helm Values

```bash
helm repo add dapr https://dapr.github.io/helm-charts/
helm repo update

helm install dapr dapr/dapr \
  --namespace dapr-system \
  --create-namespace \
  --version 1.13.0 \
  -f platform/dapr-values.yaml \
  --wait
```

Example `dapr-values.yaml`:

```yaml
global:
  ha:
    enabled: true        # High availability mode for production

dapr_operator:
  replicaCount: 2

dapr_sentry:
  replicaCount: 2

dapr_placement:
  replicaCount: 3

dapr_dashboard:
  enabled: true

prometheus:
  enabled: true
```

## Cluster RBAC for Platform Team

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: dapr-platform-admin
rules:
  - apiGroups: ["dapr.io"]
    resources: ["components", "configurations", "subscriptions", "resiliencies"]
    verbs: ["*"]
  - apiGroups: [""]
    resources: ["namespaces", "pods"]
    verbs: ["get", "list", "watch"]
```

## Upgrade Process

Document the upgrade runbook:

```bash
#!/bin/bash
# upgrade-dapr.sh
NEW_VERSION=$1
CLUSTER=$2

echo "Upgrading Dapr to $NEW_VERSION on $CLUSTER"

# 1. Check release notes for breaking changes
open "https://github.com/dapr/dapr/releases/tag/v${NEW_VERSION}"

# 2. Upgrade in staging first
helm upgrade dapr dapr/dapr \
  --namespace dapr-system \
  --version "$NEW_VERSION" \
  -f "platform/values-staging.yaml"

# 3. Run validation tests
kubectl wait --for=condition=available \
  deployment/dapr-operator \
  -n dapr-system \
  --timeout=120s

echo "Upgrade complete. Monitor error rates for 30 minutes."
```

## On-Call Rotation for Dapr Platform

Set up an on-call rotation specifically for Dapr platform issues:

```yaml
oncall_runbook:
  dapr_control_plane_down:
    - "Check dapr-operator, dapr-sentry pods"
    - "kubectl get pods -n dapr-system"
    - "Review recent Helm releases: helm history dapr -n dapr-system"
  component_init_failures:
    - "Check sidecar logs: kubectl logs POD -c daprd"
    - "Verify secret store access from the affected namespace"
    - "Validate component YAML with kubectl describe component NAME"
```

## Summary

A Dapr platform team centralizes control plane operations, component governance, security management, and developer support. Using Helm with production-grade values files for HA deployments, defining clear RBAC boundaries, and maintaining documented upgrade runbooks and on-call playbooks allows application teams to adopt Dapr confidently without needing to understand the operational details.
