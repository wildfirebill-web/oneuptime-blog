# How to Upgrade Portainer CE on Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Upgrade, Kubernetes, Helm, Updates

Description: A guide to upgrading Portainer CE deployed on Kubernetes using Helm and kubectl, covering data preservation and rollback procedures.

## Overview

Upgrading Portainer CE on Kubernetes can be done via Helm (if installed with Helm) or by updating the Kubernetes Deployment/StatefulSet manifest. This guide covers both methods and best practices for production Kubernetes Portainer upgrades.

## Method 1: Upgrade via Helm

If Portainer was installed with Helm:

```bash
# Check current Portainer Helm release

helm list -n portainer

# Update Helm repository
helm repo update portainer

# Check available versions
helm search repo portainer/portainer --versions | head -10

# Upgrade to latest version
helm upgrade portainer portainer/portainer \
  --namespace portainer \
  --reuse-values

# Or upgrade to specific version
helm upgrade portainer portainer/portainer \
  --namespace portainer \
  --version 1.0.48 \
  --reuse-values

# Monitor upgrade
kubectl rollout status deployment portainer -n portainer
```

## Method 2: Upgrade by Updating Image

For Portainer deployed via kubectl apply:

```bash
# Update image to new version
kubectl set image deployment/portainer \
  portainer=portainer/portainer-ce:latest \
  -n portainer

# Or for a specific version
kubectl set image deployment/portainer \
  portainer=portainer/portainer-ce:2.20.2 \
  -n portainer

# Monitor rollout
kubectl rollout status deployment/portainer -n portainer
```

## Method 3: Apply Updated Manifest

```bash
# Download the latest Portainer manifest
curl -o portainer-new.yaml https://downloads.portainer.io/ce2-20/portainer.yaml

# Review changes
diff portainer-current.yaml portainer-new.yaml

# Apply updated manifest
kubectl apply -f portainer-new.yaml

# Monitor upgrade
kubectl rollout status -n portainer deployment/portainer
```

## Backup Before Upgrading

```bash
# Backup Portainer data using a temporary pod
kubectl run portainer-backup \
  --image=alpine \
  --restart=Never \
  --overrides='{"spec":{"volumes":[{"name":"data","persistentVolumeClaim":{"claimName":"portainer"}}],"containers":[{"name":"backup","image":"alpine","command":["tar","czf","/backup/portainer.tar.gz","-C","/data","."],"volumeMounts":[{"name":"data","mountPath":"/data"}]}]}}' \
  -n portainer

# Wait for backup to complete
kubectl wait --for=condition=complete pod/portainer-backup -n portainer --timeout=120s

# Collect the backup
kubectl cp portainer/portainer-backup:/backup/portainer.tar.gz ./portainer-backup.tar.gz

# Clean up backup pod
kubectl delete pod portainer-backup -n portainer
```

## Rollback if Upgrade Fails

```bash
# Rollback the Kubernetes deployment
kubectl rollout undo deployment/portainer -n portainer

# Or rollback Helm release
helm rollback portainer 1 -n portainer   # Roll back to revision 1

# Check rollback history
kubectl rollout history deployment/portainer -n portainer
helm history portainer -n portainer
```

## Upgrade Portainer Agent on Kubernetes

The Portainer agent needs to be upgraded separately:

```bash
# Upgrade agent using kubectl
kubectl set image daemonset/portainer-agent \
  portainer-agent=portainer/agent:latest \
  -n portainer

# Monitor agent rollout
kubectl rollout status daemonset/portainer-agent -n portainer
```

## Verify Upgrade

```bash
# Check all Portainer pods are running
kubectl get pods -n portainer

# Check Portainer version
kubectl exec -n portainer deployment/portainer -- /portainer --version

# Check logs for errors
kubectl logs -n portainer deployment/portainer --tail=50
```

## Conclusion

Upgrading Portainer CE on Kubernetes is straightforward with either Helm or kubectl. The key is that the PersistentVolumeClaim (PVC) storing Portainer data persists through the upgrade, ensuring no configuration is lost. Always backup before upgrading, especially on production clusters. Helm provides the cleanest upgrade path with built-in rollback support.
