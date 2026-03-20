# How to Install Longhorn from Rancher UI

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Longhorn, Kubernetes, Rancher, Storage, Installation

Description: A complete walkthrough of installing Longhorn distributed storage directly from the Rancher UI using the built-in Apps & Marketplace feature.

## Introduction

Rancher provides a seamless way to install Longhorn directly from its web UI using the built-in Apps & Marketplace. Since Longhorn is a Rancher/SUSE project, it is a first-class citizen in the Rancher ecosystem with deep integration for installation, monitoring, and management. This guide walks through the Rancher UI installation process step by step.

## Prerequisites

- Rancher (version 2.5 or later) managing one or more downstream clusters
- Target Kubernetes cluster running and imported into Rancher
- Node prerequisites met on all cluster nodes:
  - `open-iscsi` installed and enabled
  - Sufficient disk space (at least 10 GiB per node)
- Appropriate Rancher RBAC permissions to install applications

### Prepare Nodes

Before installation, SSH into each cluster node and install the required packages:

```bash
# Ubuntu/Debian nodes
apt-get install -y open-iscsi nfs-common
systemctl enable --now iscsid

# RHEL/CentOS/Rocky nodes
yum install -y iscsi-initiator-utils nfs-utils
systemctl enable --now iscsid
```

## Step 1: Navigate to the Apps & Marketplace

1. Log in to your Rancher instance at `https://<rancher-url>`
2. From the top-left cluster selector, choose the target cluster where you want to install Longhorn
3. In the left sidebar, navigate to **Apps** → **Charts**

## Step 2: Find the Longhorn Chart

1. In the Charts search bar, type **Longhorn**
2. The Longhorn chart card should appear (it is included in the Rancher catalog by default)
3. Click on the **Longhorn** chart card

## Step 3: Configure the Installation

1. Click **Install** to begin the configuration wizard
2. On the **Metadata** page:
   - Select the **Namespace**: `longhorn-system` (create it if it does not exist)
   - Set the **Name** to `longhorn`
3. Click **Next** to proceed to the **Values** page

## Step 4: Customize Installation Values

The Values page presents a form-based editor for Longhorn's Helm chart values. Key settings to configure:

### Default Settings

```yaml
# These can be set in the Rancher UI form or as raw YAML values

defaultSettings:
  # Number of replicas for new volumes
  defaultReplicaCount: 3

  # Backup target (leave empty to configure later)
  backupTarget: ""

  # Storage over-provisioning percentage
  storageOverProvisioningPercentage: 200

  # Minimum available storage (%)
  storageMinimalAvailablePercentage: 25
```

### Persistence Settings

```yaml
persistence:
  # Set Longhorn as the default storage class
  defaultClass: true
  defaultClassReplicaCount: 3
  reclaimPolicy: Delete
```

### Service Configuration

```yaml
service:
  ui:
    # Use NodePort if you want to access via node IP
    type: ClusterIP
  manager:
    type: ClusterIP
```

## Step 5: Install Longhorn

1. After configuring the values, click **Install**
2. Rancher will display a progress log showing the Helm install output
3. Wait for all resources to be created — this typically takes 2–5 minutes

## Step 6: Verify the Installation

### Via Rancher UI

1. Navigate to **Workloads** → **Pods** and filter by namespace `longhorn-system`
2. Confirm all pods show **Running** status
3. Navigate to **Service Discovery** → **Services** to see the `longhorn-frontend` service

### Via kubectl

```bash
# Check all pods are running
kubectl get pods -n longhorn-system

# Check Longhorn nodes are detected
kubectl get nodes.longhorn.io -n longhorn-system

# Verify the storage class was created
kubectl get storageclass
```

## Step 7: Access the Longhorn UI from Rancher

Rancher provides a direct link to the Longhorn UI:

1. In the Rancher UI, navigate to the cluster
2. Go to **Apps** → **Installed Apps**
3. Find the `longhorn` entry and click on it
4. Look for the **Endpoints** section — click the Longhorn UI URL

Alternatively, use the direct Rancher proxy URL:

```
https://<rancher-url>/k8s/clusters/<cluster-id>/api/v1/namespaces/longhorn-system/services/http:longhorn-frontend:80/proxy/
```

## Step 8: Create a Test Volume

From the Longhorn UI:

1. Click **Volume** in the left navigation
2. Click **Create Volume**
3. Set a name, size (e.g., `1Gi`), and number of replicas
4. Click **OK**

Or from `kubectl`:

```yaml
# test-pvc-rancher.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-longhorn
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 1Gi
```

```bash
kubectl apply -f test-pvc-rancher.yaml
kubectl get pvc test-longhorn
```

## Upgrading Longhorn from Rancher UI

When a new version is available:

1. Navigate to **Apps** → **Installed Apps**
2. Find `longhorn` and click the three-dot menu
3. Select **Edit/Upgrade**
4. Choose the new chart version and click **Upgrade**

## Conclusion

Installing Longhorn from the Rancher UI combines the ease of a graphical installation wizard with the power of Helm-based configuration. Rancher's deep integration with Longhorn means you get a polished experience for installation, upgrades, and day-to-day management. With Longhorn installed, your cluster is ready to provide reliable distributed storage for stateful applications.
