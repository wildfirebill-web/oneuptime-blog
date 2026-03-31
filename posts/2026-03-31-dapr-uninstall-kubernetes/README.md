# How to Uninstall Dapr from Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Kubernetes, Uninstall, Helm, Cleanup

Description: Learn how to uninstall Dapr from a Kubernetes cluster using the Dapr CLI or Helm, and clean up all related resources including CRDs and namespaces.

---

## What Dapr Installs in Kubernetes

When you run `dapr init -k`, Dapr deploys the following control plane components in the `dapr-system` namespace:

- `dapr-operator` - manages component CRDs
- `dapr-sidecar-injector` - injects sidecar proxies
- `dapr-placement` - actor placement service
- `dapr-sentry` - certificate authority
- `dapr-scheduler` - job scheduling service

## Uninstall Using the Dapr CLI

The simplest way to remove Dapr from Kubernetes:

```bash
dapr uninstall -k
```

This removes all control plane deployments, services, and the `dapr-system` namespace.

## Uninstall a Specific Namespace

If you installed Dapr in a custom namespace:

```bash
dapr uninstall -k --namespace my-dapr-namespace
```

## Uninstall Using Helm

If you installed Dapr via Helm, uninstall using Helm:

```bash
helm uninstall dapr -n dapr-system
```

Remove the namespace after:

```bash
kubectl delete namespace dapr-system
```

## Removing CRDs

By default, Dapr CRDs are not removed to protect your component configurations. To explicitly delete them:

```bash
kubectl get crds | grep dapr.io | awk '{print $1}' | xargs kubectl delete crd
```

## Removing Component Resources

Before removing CRDs, delete your component, subscription, and configuration resources:

```bash
kubectl delete components --all -n default
kubectl delete subscriptions.dapr.io --all -n default
kubectl delete configurations.dapr.io --all -n default
```

## Removing Sidecar Annotations

After uninstalling Dapr, remove sidecar annotations from your deployments to prevent injection errors:

```yaml
# Remove these annotations from your deployment spec
# dapr.io/enabled: "true"
# dapr.io/app-id: "myapp"
```

```bash
kubectl annotate deployment myapp dapr.io/enabled-
```

## Verifying Removal

```bash
# Check no dapr pods remain
kubectl get pods -n dapr-system

# Check CRDs are removed
kubectl get crds | grep dapr

# Check no dapr services remain
kubectl get svc -n dapr-system
```

## Summary

Uninstall Dapr from Kubernetes using `dapr uninstall -k` or `helm uninstall dapr`. CRDs are preserved by default - delete them manually if needed. Also remove component resources and sidecar annotations from your application deployments to ensure a clean removal.
