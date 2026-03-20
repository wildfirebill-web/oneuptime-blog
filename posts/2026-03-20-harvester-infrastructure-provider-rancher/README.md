# How to Use Harvester as Infrastructure Provider in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Harvester, Kubernetes, Rancher, Virtualization, HCI, Infrastructure

Description: Learn how to configure and use Harvester as an infrastructure provider in Rancher for automated Kubernetes cluster provisioning on VM infrastructure.

## Introduction

When Harvester is registered as an infrastructure provider in Rancher, it enables Rancher to automatically provision virtual machines in Harvester and use them as nodes for Kubernetes clusters. This creates an automated pipeline where requesting a new Kubernetes cluster automatically triggers VM creation in Harvester, OS installation, and Kubernetes bootstrap — all without manual intervention.

## Prerequisites

- Harvester cluster imported into Rancher
- Rancher 2.7.0 or higher
- The Harvester node driver enabled in Rancher
- VM images available in Harvester
- Cloud credentials configured in Rancher

## Step 1: Verify the Harvester Node Driver

```bash
# Check if the Harvester node driver is active in Rancher
kubectl get nodedriver harvester -n cattle-system

# Via Rancher API
curl -sk -H "Authorization: Bearer $RANCHER_TOKEN" \
    https://rancher.company.com/v3/nodedrivers | \
    jq '.data[] | select(.name == "harvester") | {name: .name, state: .state, active: .active}'
```

If not enabled:
1. In Rancher, go to **Cluster Management** → **Drivers**
2. Click **Node Drivers**
3. Find **Harvester** and click **Activate**

## Step 2: Create Cloud Credentials for Harvester

Cloud credentials store the connection information for Rancher to communicate with Harvester:

### Via Rancher UI

1. Navigate to **Cluster Management** → **Cloud Credentials**
2. Click **Create**
3. Select **Harvester**
4. Configure:

```
Name: harvester-prod-creds
Harvester Cluster: local-harvester  (select from dropdown)
```

5. Click **Create**

### Via Rancher API

```bash
# Create Harvester cloud credentials via API

RANCHER_URL="https://rancher.company.com"
RANCHER_TOKEN="token-xxxxx:xxxxxx"

# First, get the Harvester cluster ID
HARVESTER_CLUSTER_ID=$(curl -sk \
    -H "Authorization: Bearer ${RANCHER_TOKEN}" \
    "${RANCHER_URL}/v3/clusters" | \
    jq -r '.data[] | select(.labels["provider.cattle.io"] == "harvester") | .id')

echo "Harvester cluster ID: ${HARVESTER_CLUSTER_ID}"

# Create the cloud credential
curl -sk -X POST \
    -H "Authorization: Bearer ${RANCHER_TOKEN}" \
    -H "Content-Type: application/json" \
    "${RANCHER_URL}/v3/cloudcredentials" \
    -d "{
        \"type\": \"cloudCredential\",
        \"name\": \"harvester-prod-creds\",
        \"harvesterCredentialConfig\": {
            \"clusterId\": \"${HARVESTER_CLUSTER_ID}\",
            \"clusterType\": \"imported\"
        }
    }"
```

## Step 3: Configure a Node Template

Node templates define the VM specifications that Rancher will use when provisioning nodes:

### Via Rancher UI

1. Go to **Cluster Management** → **Node Templates**
2. Click **Create**
3. Select **Harvester**
4. Configure the VM specification:

```
Template Name:  ubuntu-22-04-large
Cloud Creds:    harvester-prod-creds
Cluster:        local-harvester
Namespace:      default
Image Name:     ubuntu-22-04-lts
Network:        default/vlan-100
CPU Count:      8
Memory:         16 GB
Disk Size:      100 GB
SSH User:       ubuntu
```

### Via kubectl (RKE2 Machine Config)

```yaml
# harvester-machine-config.yaml
# Machine configuration for Rancher-provisioned nodes on Harvester

apiVersion: rke-machine-config.cattle.io/v1
kind: HarvesterConfig
metadata:
  name: ubuntu-large-node
  namespace: fleet-default
spec:
  # Harvester cluster name (matches imported cluster in Rancher)
  clusterName: local-harvester
  # Harvester namespace where VMs will be created
  namespace: default
  # Image for the VM
  imageName: default/ubuntu-22-04-lts
  # Network for the VM
  networkName: default/vlan-100
  # VM size
  cpuCount: "8"
  memorySize: "16"
  # Root disk
  diskSize: "100"
  diskStorageClassName: longhorn
  # SSH user for Rancher to access during bootstrap
  sshUser: ubuntu
  # Cloud-init for node preparation
  userData: |
    #cloud-config
    package_update: true
    packages:
      - qemu-guest-agent
      - curl
      - open-iscsi
    runcmd:
      - systemctl enable --now qemu-guest-agent
      - systemctl enable --now iscsid
      # Required for Longhorn in guest cluster
      - modprobe iscsi_tcp
      - echo 'iscsi_tcp' >> /etc/modules-load.d/iscsi.conf
```

## Step 4: Provision a Kubernetes Cluster Using Harvester

Now provision a new cluster using Harvester as the infrastructure:

```yaml
# app-cluster-on-harvester.yaml
# Production application cluster provisioned on Harvester infrastructure

apiVersion: provisioning.cattle.io/v1
kind: Cluster
metadata:
  name: app-cluster-prod
  namespace: fleet-default
  labels:
    environment: production
    infrastructure: harvester
spec:
  rkeConfig:
    machinePools:
      # 3-node control plane
      - name: control-plane
        quantity: 3
        etcdRole: true
        controlPlaneRole: true
        workerRole: false
        machineConfigRef:
          kind: HarvesterConfig
          name: ubuntu-small-control-plane
      # Auto-scaling worker pool
      - name: workers
        quantity: 3
        # Enable auto-scaling
        machineDeploymentAnnotations:
          cluster.x-k8s.io/cluster-api-autoscaler-node-group-min-size: "3"
          cluster.x-k8s.io/cluster-api-autoscaler-node-group-max-size: "10"
        etcdRole: false
        controlPlaneRole: false
        workerRole: true
        machineConfigRef:
          kind: HarvesterConfig
          name: ubuntu-large-node
    # Kubernetes component configuration
    machineGlobalConfig:
      # Disable default ingress
      disable-cloud-provider: false
    etcd:
      # etcd snapshot configuration
      snapshotRetention: 5
      snapshotScheduleCron: "0 */6 * * *"
  kubernetesVersion: "v1.27.9+rke2r1"
  networkConfig:
    plugin: canal
  # Cloud provider configuration for Harvester
  cloudProviderConfig: |
    [Global]
    [LoadBalancer]
    lb-namespace=kube-system
    lb-image-version=v0.3.0
```

```bash
kubectl apply -f harvester-machine-config.yaml
kubectl apply -f app-cluster-on-harvester.yaml

# Monitor provisioning
kubectl get cluster app-cluster-prod -n fleet-default -w

# Watch the VMs being created in Harvester
kubectl get vmi -n default | grep app-cluster
```

## Step 5: Verify Provisioned Infrastructure

```bash
# Check the machines (VMs) created by Rancher
kubectl get machines -n fleet-default \
    -l cluster.x-k8s.io/cluster-name=app-cluster-prod

# In the Harvester cluster, see the VMs
kubectl get vmi -n default | grep app-cluster

# Access the new cluster
kubectl get secret app-cluster-prod-kubeconfig -n fleet-default \
    -o jsonpath='{.data.value}' | base64 -d > app-cluster.kubeconfig

export KUBECONFIG=app-cluster.kubeconfig
kubectl get nodes
```

## Automating Cluster Provisioning with Fleet

Use Rancher Fleet for GitOps-driven cluster provisioning:

```yaml
# fleet.yaml
# Deploy this cluster definition via Fleet

defaultNamespace: fleet-default
helm:
  chart: ./charts/app-cluster
  valuesFiles:
    - values-prod.yaml

# This file goes in a Git repo that Fleet watches
# When committed, Fleet automatically creates the cluster in Rancher/Harvester
```

## Conclusion

Using Harvester as an infrastructure provider in Rancher enables fully automated, API-driven Kubernetes cluster provisioning on VM infrastructure. This combination is powerful for platform engineering teams that need to provide self-service Kubernetes clusters to development teams. The entire process — from requesting a cluster to having nodes provisioned in Harvester and Kubernetes bootstrapped — can be driven by a simple API call or a GitOps commit, dramatically reducing the time from cluster request to usable environment.
