# How to Manage Harvester from Rancher Dashboard

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Harvester, Kubernetes, Rancher, Virtualization, HCI, Dashboard

Description: Learn how to manage your Harvester HCI cluster, virtual machines, and infrastructure from the Rancher unified management dashboard.

## Introduction

Once Harvester is integrated with Rancher, you can access all Harvester management functionality directly from the Rancher dashboard. This eliminates the need to switch between interfaces and provides a unified view of your entire infrastructure — both VMs and Kubernetes clusters. This guide covers the key Harvester management tasks available from Rancher.

## Accessing Harvester from Rancher

### Navigate to the Harvester Cluster

1. Log into the Rancher dashboard at `https://rancher.company.com`
2. In the top navigation, click the cluster selector
3. Select your Harvester cluster (e.g., `local-harvester`)
4. You are now in the Harvester management context within Rancher

Alternatively, from the main dashboard:
1. Click **Cluster Management**
2. Find the Harvester cluster in the list
3. Click **Explore** to open the cluster management view

## Managing Virtual Machines

### View All VMs

From the Harvester context in Rancher:

1. Click **Virtualization Management** in the left sidebar
2. Select **Virtual Machines**
3. You see all VMs with their status, resource usage, and node placement

```bash
# Equivalent kubectl command
export KUBECONFIG=harvester.kubeconfig
kubectl get virtualmachineinstances -A \
    -o custom-columns=\
'NAME:.metadata.name,NS:.metadata.namespace,STATE:.status.phase,NODE:.status.nodeName,IP:.status.interfaces[0].ipAddress'
```

### Create a VM from Rancher

1. Click **Create** in the Virtual Machines view
2. The Harvester VM creation wizard opens within Rancher
3. Fill in the same fields as in the native Harvester UI:
   - Name and namespace
   - CPU and memory
   - Boot image
   - Network
   - Cloud-init

### Monitor VM Metrics

From the VM details page in Rancher:
1. Click on a VM name
2. Go to the **Metrics** tab
3. View CPU usage, memory usage, network I/O, and disk I/O

## Managing VM Images

### Import Images via Rancher

1. Navigate to **Virtualization Management** → **Images**
2. Click **Create**
3. Provide the image URL and name
4. Click **Create**

```bash
# Monitor image import progress
kubectl get virtualmachineimage -n default -w
```

### Manage Image Lifecycle

From the images list in Rancher:
- **Delete**: Remove unused images
- **Download**: Export an image for offline use
- **Copy URL**: Get the download URL for scripting

## Managing Volumes and Storage

### View Volumes

1. Navigate to **Virtualization Management** → **Volumes**
2. See all PVCs with size, storage class, and attachment status

### Create a Volume

1. Click **Create**
2. Set name, namespace, size, and storage class
3. Click **Create**

## Managing VM Networks

### View Networks

1. Navigate to **Networks** → **VM Networks**
2. See all NetworkAttachmentDefinitions with their VLAN IDs

### Create a VLAN Network

1. Click **Create**
2. Provide:
   - Name and namespace
   - Cluster network
   - VLAN ID
3. Click **Create**

## Managing Cluster Nodes

### View Node Status

1. Navigate to **Hosts** in Harvester (or **Infrastructure** → **Nodes**)
2. See each node with CPU, memory, and storage usage

```bash
# Via kubectl
kubectl get nodes -o wide
kubectl top nodes
```

### Cordon/Uncordon a Node

From the nodes list:
1. Click the **⋮** menu on a node
2. Select **Cordon** to prevent new VMs from scheduling
3. Select **Uncordon** to re-enable scheduling

```bash
# Via kubectl
kubectl cordon harvester-node-02
kubectl uncordon harvester-node-02
```

## Managing Rancher Clusters on Harvester

One of the most powerful features of the Rancher-Harvester integration is cluster provisioning:

### Provision a New Kubernetes Cluster

1. In Rancher, navigate to **Cluster Management**
2. Click **Create**
3. Select **RKE2/K3s** and choose **Harvester** as the provider
4. Configure node pools with Harvester VM specifications
5. Click **Create**

Rancher will automatically:
- Create VMs in Harvester using the Harvester node driver
- Bootstrap RKE2 or K3s on the VMs
- Register the cluster with Rancher
- Configure the Harvester CSI driver for persistent storage

### View Cluster Health Dashboard

For clusters running on Harvester:

1. Click on the cluster name in Rancher
2. View the cluster dashboard:
   - Node count and health
   - CPU and memory utilization
   - Pod count
   - Deployment health

### Scale Node Pools

1. Click on the cluster
2. Go to **Cluster** → **Machine Pools**
3. Click **Edit** on a pool
4. Change the quantity
5. Click **Save** — new VMs will be created in Harvester automatically

## Using Rancher's Monitoring in Harvester Context

Rancher's Prometheus/Grafana stack can be deployed to monitor Harvester:

```bash
# Install Rancher Monitoring on the Harvester cluster
# Via Rancher UI:
# 1. Navigate to the Harvester cluster
# 2. Click Apps → Charts
# 3. Find "Monitoring"
# 4. Click Install

# Or via Helm:
helm repo add rancher-charts https://charts.rancher.io
helm upgrade --install rancher-monitoring rancher-charts/rancher-monitoring \
    --namespace cattle-monitoring-system \
    --create-namespace \
    --set prometheus.prometheusSpec.retentionSize=10GiB
```

## Managing RBAC from Rancher

Rancher provides centralized RBAC for Harvester:

### Create a Harvester Project

1. Navigate to the Harvester cluster in Rancher
2. Go to **Cluster** → **Projects/Namespaces**
3. Click **Create Project**
4. Assign team members with appropriate roles

### Assign VM Management Roles

```yaml
# rancher-harvester-role.yaml
# Custom role for VM operators

apiVersion: management.cattle.io/v3
kind: GlobalRole
metadata:
  name: vm-operator
rules:
  - apiGroups: ["kubevirt.io"]
    resources: ["virtualmachines", "virtualmachineinstances"]
    verbs: ["get", "list", "create", "update", "delete"]
  - apiGroups: ["harvesterhci.io"]
    resources: ["virtualmachineimages"]
    verbs: ["get", "list"]
```

## Conclusion

Managing Harvester through the Rancher dashboard provides a significantly improved operational experience compared to using each tool separately. The unified interface reduces context switching, provides consistent RBAC across both VM and container workloads, and makes it straightforward to provision new Kubernetes clusters on your HCI infrastructure. As your organization grows, the Rancher-Harvester integration scales with you — supporting multi-cluster management, fleet deployments, and policy enforcement across your entire hybrid infrastructure.
