# How to Safely Uninstall Longhorn from Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Longhorn, Uninstall, Kubernetes, Storage, Data Migration, SUSE Rancher, Cleanup

Description: Learn how to safely uninstall Longhorn from a Kubernetes cluster, including migrating data off Longhorn volumes, removing workloads, and cleaning up CRDs and storage data.

---

Uninstalling Longhorn requires careful preparation to avoid data loss. You must migrate data off Longhorn volumes, remove workloads using those volumes, and then follow Longhorn's official uninstall procedure.

---

## Pre-Uninstall Checklist

- [ ] Back up all Longhorn volume data to an external store
- [ ] Migrate stateful workloads to alternative storage (hostPath, NFS, cloud)
- [ ] Delete all PVCs backed by Longhorn
- [ ] Ensure no pods are mounted to Longhorn volumes

---

## Step 1: Back Up All Data

Before uninstalling, back up critical volumes:

```bash
# Trigger backup of all volumes to S3/NFS backup target

# Via Longhorn UI: each volume > Create Backup
# Or via API:
for vol in $(kubectl get lhvolume -n longhorn-system -o name); do
  name=$(echo $vol | cut -d/ -f2)
  echo "Backing up $name..."
  kubectl annotate lhvolume $name -n longhorn-system \
    "recurring-jobs.longhorn.io/source=backup"
done
```

---

## Step 2: Scale Down Workloads and Delete PVCs

```bash
# Scale down all deployments using Longhorn PVCs
kubectl get pvc -A | grep longhorn | while read ns name rest; do
  # Find pods using this PVC
  kubectl get pods -n $ns -o json | \
    jq -r '.items[] | select(.spec.volumes[].persistentVolumeClaim.claimName == "'$name'") | .metadata.name'
done

# Scale down identified deployments
kubectl scale deployment <name> --replicas=0 -n <namespace>

# Delete PVCs after workloads are stopped
kubectl delete pvc --all -n my-app
```

---

## Step 3: Enable Longhorn Uninstall Mode

Longhorn has a deletion confirmation mechanism. Set the uninstall flag:

```bash
# Set the allow-node-drain-with-last-healthy-replica setting first
kubectl patch setting.longhorn.io deleting-confirmation-flag \
  -n longhorn-system \
  --type merge \
  -p '{"value":"true"}'
```

---

## Step 4: Uninstall Longhorn via Helm

```bash
# If installed via Helm
helm uninstall longhorn -n longhorn-system

# Wait for all Longhorn pods to terminate
kubectl get pods -n longhorn-system -w
```

---

## Step 5: Remove Longhorn CRDs

```bash
# Remove all Longhorn CRDs
kubectl get crds | grep longhorn | awk '{print $1}' | \
  xargs kubectl delete crd

# Remove the namespace
kubectl delete namespace longhorn-system
```

---

## Step 6: Clean Up Node Data

On each storage node, remove Longhorn data directories:

```bash
#!/bin/bash
# run on each node
rm -rf /var/lib/longhorn
# If using custom data path, adjust accordingly
```

---

## Troubleshooting Stuck Uninstall

If CRDs are stuck in `Terminating`:

```bash
# Remove finalizers from stuck Longhorn resources
for crd in $(kubectl get crds -o name | grep longhorn); do
  kubectl patch $crd --type json \
    -p '[{"op":"remove","path":"/metadata/finalizers"}]'
done
```

---

## Best Practices

- Never delete the `longhorn-system` namespace directly - always use the proper uninstall procedure.
- Verify all backups are accessible in the backup store before uninstalling.
- After uninstalling, verify that node-local Longhorn data is deleted to reclaim disk space.
