# How to Fix "Storage Class Detection Error" in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Troubleshooting, Kubernetes, Storage Class, StorageClass, CSI

Description: Learn how to fix "Storage Class Detection Error" in Portainer Kubernetes environments by configuring default StorageClasses and resolving CSI driver issues.

---

The "Storage Class Detection Error" in Portainer appears when managing Kubernetes environments and Portainer cannot enumerate the available storage classes. This affects the ability to create PersistentVolumeClaims from the Portainer UI.

## Step 1: Check Storage Classes in Kubernetes

```bash
# List all storage classes in the cluster
kubectl get storageclasses

# Check if a default storage class is set
kubectl get storageclasses -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.annotations.storageclass\.kubernetes\.io/is-default-class}{"\n"}{end}'
```

## Step 2: Set a Default Storage Class

If no default storage class is set, Portainer cannot auto-detect which to use:

```bash
# Mark a storage class as default (replace 'local-path' with your storage class name)
kubectl patch storageclass local-path \
  -p '{"metadata": {"annotations": {"storageclass.kubernetes.io/is-default-class": "true"}}}'

# Verify
kubectl get storageclasses
# The default class should show "(default)" next to its name
```

## Step 3: Install a Default Storage Class

For bare-metal Kubernetes clusters without a CSI driver, install one:

```bash
# Option 1: Rancher Local Path Provisioner (good for single-node)
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml

# Option 2: NFS Subdir External Provisioner (for NFS)
helm repo add nfs-subdir-external-provisioner \
  https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/

helm install nfs-subdir-external-provisioner \
  nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --set nfs.server=192.168.1.100 \
  --set nfs.path=/mnt/nfs_share
```

## Step 4: Check Portainer RBAC Permissions

Portainer needs permission to read storage classes:

```bash
# Check if the Portainer service account can list storage classes
kubectl auth can-i list storageclasses --as=system:serviceaccount:portainer:portainer-sa

# If "no", create the required ClusterRole binding
kubectl create clusterrolebinding portainer-storage \
  --clusterrole=view \
  --serviceaccount=portainer:portainer-sa
```

## Step 5: Refresh Kubernetes Environment in Portainer

After fixing the storage class:

1. In Portainer go to **Environments > Kubernetes environment**.
2. Click **Edit** and then **Update environment**.
3. Portainer will re-enumerate cluster resources including storage classes.

## Step 6: Check CSI Driver Pods

If using a CSI driver, verify its pods are healthy:

```bash
# Check CSI driver pods are running
kubectl get pods -n kube-system | grep csi

# Check CSI driver logs for errors
kubectl logs -n kube-system <csi-pod-name>
```
