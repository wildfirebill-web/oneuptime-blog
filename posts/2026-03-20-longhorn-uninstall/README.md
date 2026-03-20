# How to Uninstall Longhorn Safely

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Longhorn, Uninstall, Kubernetes, Storage, Cleanup, SUSE Rancher

Description: Learn how to safely uninstall Longhorn from a Kubernetes cluster by migrating workloads, removing volumes, and cleanly removing all Longhorn resources and system components.

---

Uninstalling Longhorn without following the correct procedure can leave orphaned resources, blocked namespaces, or corrupted PVCs. Always migrate data before uninstalling.

---

## Before You Begin

**Warning**: Uninstalling Longhorn deletes all Longhorn-managed volumes and their data. Ensure you have:
- Migrated all application data to an alternative storage solution
- Created backups of any Longhorn volumes you want to keep
- Scaled down all workloads using Longhorn volumes

---

## Step 1: Scale Down All Workloads

```bash
# Find all workloads using Longhorn PVCs
kubectl get pvc -A -o json | \
  jq -r '.items[] | select(.spec.storageClassName | startswith("longhorn")) | "\(.metadata.namespace)/\(.metadata.name)"'

# Scale down each deployment/statefulset using Longhorn volumes
kubectl scale deployment <name> -n <namespace> --replicas=0
kubectl scale statefulset <name> -n <namespace> --replicas=0
```

---

## Step 2: Delete All PVCs and PVs

```bash
# Delete PVCs using Longhorn storage classes
kubectl get pvc -A | grep longhorn

# Delete each PVC (this also deletes the underlying Longhorn volume)
kubectl delete pvc <pvc-name> -n <namespace>

# Verify no Longhorn volumes remain
kubectl get volume -n longhorn-system
```

---

## Step 3: Enable Longhorn Uninstallation Setting

Longhorn requires you to explicitly allow uninstallation to prevent accidental removal:

```bash
# Enable the allow-recurring-job-while-volume-detached setting
kubectl -n longhorn-system patch -p '{"value": "true"}' \
  --type=merge lhs deleting-confirmation-flag

# Or via Longhorn UI: Settings → General → Deleting Confirmation Flag → true
```

---

## Step 4: Uninstall Longhorn via Helm

```bash
# Uninstall using Helm
helm uninstall longhorn -n longhorn-system

# Monitor the deletion of Longhorn pods
kubectl get pods -n longhorn-system -w
```

---

## Step 5: Uninstall Longhorn via Rancher UI

If Longhorn was installed through Rancher Apps & Marketplace:

1. Navigate to **Apps & Marketplace** → **Installed Apps**
2. Select **Longhorn**
3. Click **Delete**
4. Wait for all Longhorn pods to terminate

---

## Step 6: Remove the Longhorn Namespace

After the Helm uninstall, verify the namespace is clean:

```bash
# Check if any resources remain
kubectl get all -n longhorn-system
kubectl get crd | grep longhorn

# If the namespace is stuck in Terminating state, finalize it
kubectl get namespace longhorn-system -o json | \
  jq '.spec.finalizers = []' | \
  kubectl replace --raw /api/v1/namespaces/longhorn-system/finalize -f -
```

---

## Step 7: Remove Longhorn CRDs

```bash
# List all Longhorn CRDs
kubectl get crd | grep longhorn

# Delete all Longhorn CRDs
kubectl get crd | grep longhorn | awk '{print $1}' | xargs kubectl delete crd
```

---

## Step 8: Clean Up Node Directories

On each cluster node, remove the Longhorn data directory:

```bash
# Run on each node
rm -rf /var/lib/longhorn/

# If using a custom data path, clean that directory instead
# The path is configured in Longhorn settings → Default Data Path
```

---

## Step 9: Remove Longhorn's Device Mapper Devices

```bash
# On each node, check for leftover device mapper devices
ls /dev/mapper/ | grep longhorn

# Remove them
dmsetup remove /dev/mapper/<longhorn-device-name>

# Also remove any leftover iSCSI sessions
iscsiadm --mode node --logout
```

---

## Step 10: Verify Clean Removal

```bash
# No Longhorn pods should be running
kubectl get pods -n longhorn-system

# No Longhorn CRDs should exist
kubectl get crd | grep longhorn

# No Longhorn namespace should exist
kubectl get namespace longhorn-system

# Verify PVs are gone
kubectl get pv | grep longhorn
```

---

## Best Practices

- Always migrate data before uninstalling — Longhorn does not preserve volume data after uninstallation.
- Use Velero or another backup tool to backup all Longhorn volumes before starting the uninstall process.
- If uninstalling from a production cluster, schedule uninstallation during a maintenance window and test the procedure in a staging environment first.
