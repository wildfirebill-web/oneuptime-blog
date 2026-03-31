# How to Use the dapr upgrade Command

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, CLI, Upgrade, Kubernetes, Version Management

Description: Learn how to use the dapr upgrade command to upgrade the Dapr runtime on Kubernetes or in self-hosted mode to a newer version.

---

## Overview

The `dapr upgrade` command upgrades the Dapr control plane components on a Kubernetes cluster or the Dapr runtime binaries in self-hosted mode. It performs a rolling upgrade of all control plane pods while preserving your component and configuration resources.

## Upgrading Dapr on Kubernetes

Upgrade to the latest stable version:

```bash
dapr upgrade --kubernetes
```

Upgrade to a specific version:

```bash
dapr upgrade --kubernetes --runtime-version 1.14.0
```

## Waiting for the Upgrade to Complete

Use `--wait` to block until all pods are running the new version:

```bash
dapr upgrade --kubernetes --runtime-version 1.14.0 --wait --timeout 300
```

## Upgrading in Self-Hosted Mode

```bash
dapr upgrade --runtime-version 1.14.0
```

This downloads and replaces the Dapr sidecar binary. Restart your applications to pick up the new version.

## Pre-Upgrade Checklist

Before running the upgrade:

1. Check current version:

```bash
dapr status --kubernetes
```

2. Review the changelog for breaking changes at https://github.com/dapr/dapr/releases

3. Test the upgrade in a staging environment first

4. Back up your component and configuration resources:

```bash
kubectl get components -n default -o yaml > components-backup.yaml
kubectl get configurations -n default -o yaml > configs-backup.yaml
```

## Upgrading with a Custom Image Registry

For air-gapped environments:

```bash
dapr upgrade --kubernetes \
             --runtime-version 1.14.0 \
             --image-registry myregistry.example.com/dapr \
             --wait
```

## Verifying the Upgrade

After the upgrade completes, confirm all components are on the new version:

```bash
dapr status --kubernetes
```

Expected output:

```text
  NAME                   NAMESPACE    HEALTHY  STATUS    VERSION
  dapr-operator          dapr-system  True     Running   1.14.0
  dapr-placement-server  dapr-system  True     Running   1.14.0
  dapr-sentry            dapr-system  True     Running   1.14.0
  dapr-sidecar-injector  dapr-system  True     Running   1.14.0
```

## Rolling Back if Needed

If the upgrade causes issues, downgrade by specifying the previous version:

```bash
dapr upgrade --kubernetes --runtime-version 1.13.0 --wait
```

## Summary

`dapr upgrade` handles the complexity of upgrading the Dapr control plane with a single command. Always use `--wait` in automated pipelines to prevent workloads from starting before the control plane is ready. Test upgrades in staging first and keep the previous version number handy for quick rollbacks.
