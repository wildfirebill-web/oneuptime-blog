# How to Configure vSphere Cloud Provider in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, vSphere, VMware, Cloud Provider

Description: Configure the vSphere cloud provider in Rancher to enable dynamic VMware datastore volumes and vSphere load balancer integration for on-premises clusters.

## Introduction

The VMware vSphere cloud provider enables Kubernetes clusters running on vSphere to dynamically provision VMDK-backed PersistentVolumes and integrate with NSX-T or vSphere with Tanzu for load balancing. This guide covers configuring the vSphere cloud provider for RKE2 clusters managed by Rancher in an on-premises vSphere environment.

## Prerequisites

- vSphere 6.7 Update 3 or later (vSphere 7+ recommended)
- Rancher managing an RKE2 cluster on vSphere VMs
- A vSphere user account with required permissions
- All VMs configured with VMware Tools installed and UUID enabled

## Step 1: Configure vSphere VM Prerequisites

```bash
# Enable disk UUID on VMs (required for vSphere Cloud Provider)

# Run from vSphere CLI or govc on each VM:
govc vm.change -vm "/<datacenter>/vm/<vm-name>" \
  -e "disk.enableUUID=true"

# Verify
govc vm.info -vm "/<datacenter>/vm/<vm-name>" \
  | grep "disk.enableUUID"

# Install VMware Tools on each node
# For Ubuntu:
sudo apt-get install -y open-vm-tools
sudo systemctl enable --now open-vm-tools
```

## Step 2: Create a vSphere Role with Required Privileges

In the vSphere Web Client:

1. Navigate to **Administration → Access Control → Roles**.
2. Create a new role `RancherKubernetesRole` with these privileges:
   - **Datastore**: Allocate space, Browse, Low level file ops, Remove file
   - **Network**: Assign network
   - **Resource**: Assign VM to resource pool
   - **Virtual Machine → Configuration**: Add disk, Remove disk
   - **Virtual Machine → Provisioning**: Allow disk access, Allow read-only disk access
   - **vSAN**: Cluster, Datastore management

3. Assign this role to the vSphere user on the datacenter and datastore.

## Step 3: Create the vSphere Cloud Config

```ini
# /etc/rancher/rke2/vsphere-cloud-config.conf
[Global]
# vCenter server hostname or IP
server = vcenter.example.com
port = 443
user = rancher-k8s@vsphere.local
password = "SecurePassword123!"
insecure-flag = false          # Set to true if using a self-signed cert
datacenters = /Datacenter1

[VirtualCenter "vcenter.example.com"]
# Optionally override per-vCenter settings
user = rancher-k8s@vsphere.local
password = "SecurePassword123!"
datacenters = /Datacenter1

[Workspace]
server = vcenter.example.com
datacenter = /Datacenter1
default-datastore = datastore1
resourcepool-path = /Datacenter1/host/Cluster1/Resources/KubernetesPool
folder = /Datacenter1/vm/kubernetes-vms

[Disk]
scsicontrollertype = pvscsi

[Network]
public-network = "VM Network"
```

```bash
# Place the config on all nodes
sudo cp vsphere-cloud-config.conf /etc/rancher/rke2/vsphere-cloud-config.conf
sudo chmod 600 /etc/rancher/rke2/vsphere-cloud-config.conf
```

## Step 4: Configure RKE2 to Use vSphere Provider

```yaml
# /etc/rancher/rke2/config.yaml (all nodes)
cloud-provider-name: vsphere
cloud-provider-config: /etc/rancher/rke2/vsphere-cloud-config.conf
```

Restart RKE2 after the configuration change:

```bash
sudo systemctl restart rke2-server  # control-plane nodes
sudo systemctl restart rke2-agent   # worker nodes
```

## Step 5: Configure via Rancher UI

1. Navigate to **Cluster Management** → select the cluster → **⋮ → Edit Config**.
2. Under **Cloud Provider**, select **vSphere**.
3. Fill in:
   - vCenter hostname/IP
   - Username and password
   - Datacenter path
   - Datastore path
   - VM folder path
4. Click **Save**.

## Step 6: Install the vSphere CSI Driver

```bash
# Create the vSphere credentials secret
kubectl create secret generic vsphere-config-secret \
  --from-file=csi-vsphere.conf=/etc/rancher/rke2/vsphere-cloud-config.conf \
  -n kube-system

# Install vSphere CSI Driver via Helm
helm repo add vsphere-csi https://kubernetes-sigs.github.io/vsphere-csi-driver/charts
helm repo update

helm install vsphere-csi vsphere-csi/vsphere-csi-driver \
  --namespace kube-system \
  --set config.existingSecret=vsphere-config-secret
```

## Step 7: Create vSphere StorageClasses

```yaml
# vsphere-storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: vsphere-vmdk
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: csi.vsphere.volume
parameters:
  storagepolicyname: "vSAN Default Storage Policy"  # or your custom policy
  datastoreurl: "ds:///vmfs/volumes/<datastore-id>/"
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

```bash
kubectl apply -f vsphere-storageclass.yaml
kubectl get storageclass
```

## Step 8: Verify the Integration

```bash
# Test dynamic volume provisioning
kubectl apply -f - << 'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: vsphere-pvc-test
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: vsphere-vmdk
  resources:
    requests:
      storage: 10Gi
EOF

# Check PVC status - should transition to Bound
kubectl get pvc vsphere-pvc-test -w

# Verify the VMDK was created in vCenter
# (Check the Datastore browser in vSphere Web Client)
```

## Common Issues

| Issue | Resolution |
|---|---|
| `PVC stuck in Pending` | Check CSI driver pods and vCenter connectivity |
| `Unable to find VM` | Ensure VM folder path is correct in cloud config |
| `disk.enableUUID not set` | Set disk UUID on all VMs before provisioning |
| `certificate verify failed` | Set `insecure-flag = true` or import vCenter CA cert |

## Conclusion

The vSphere cloud provider brings Kubernetes-native storage provisioning to on-premises vSphere environments. With the vSphere CSI Driver and properly configured permissions, RKE2 clusters managed by Rancher can dynamically provision VMDK-backed PersistentVolumes directly from Kubernetes PVC requests, eliminating manual storage provisioning workflows.
