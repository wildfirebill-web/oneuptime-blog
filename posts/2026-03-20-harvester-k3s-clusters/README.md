# How to Create K3s Clusters on Harvester

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Harvester, Kubernetes, K3s, Rancher, Virtualization, HCI

Description: Learn how to provision lightweight K3s Kubernetes clusters on Harvester virtual machines for development, edge, and resource-constrained workloads.

## Introduction

K3s is a lightweight, certified Kubernetes distribution that requires significantly less resources than full Kubernetes. When combined with Harvester as the infrastructure provider, K3s clusters can be quickly provisioned on VMs for development environments, edge computing workloads, or any scenario where a full RKE2 cluster would be overkill. Rancher's K3s provisioning via the Harvester node driver makes this a seamless experience.

## K3s vs RKE2 on Harvester

| Feature | K3s | RKE2 |
|---|---|---|
| Resource overhead | Very low (~500 MB RAM) | Low (~1 GB RAM) |
| Use case | Dev, edge, IoT | Production, compliance |
| FIPS compliance | No | Yes |
| CIS hardening | Limited | Full support |
| Setup complexity | Minimal | Moderate |

## Prerequisites

- Harvester cluster integrated with Rancher
- Harvester cloud credentials configured in Rancher
- VM images available in Harvester
- A network configured for the K3s cluster VMs

## Step 1: Create K3s Cluster via Rancher UI

1. In Rancher, navigate to **Cluster Management** → **Clusters**
2. Click **Create**
3. Select **RKE2/K3s** (K3s is listed here)
4. Select **Harvester** as the driver
5. Toggle to **K3s** instead of RKE2

### Cluster Configuration

```text
Cluster Name:       dev-k3s-cluster
Kubernetes:         K3s v1.27.x (latest)
CNI:                Flannel (K3s default, lighter weight)
```

### Node Pool for K3s

For a simple dev cluster, a combined control-plane/worker node works:

```text
Pool Name:          all-in-one
Count:              1 (or 3 for HA)
Roles:              etcd, Control Plane, Worker
VM CPU:             2 cores
VM Memory:          4 GB
VM Image:           ubuntu-22-04-lts
VM Disk Size:       30 GB
```

## Step 2: Create K3s Cluster via Rancher API

```yaml
# k3s-harvester-cluster.yaml

# K3s cluster on Harvester for development workloads

apiVersion: provisioning.cattle.io/v1
kind: Cluster
metadata:
  name: dev-k3s-cluster
  namespace: fleet-default
  annotations:
    field.cattle.io/description: "Development K3s cluster on Harvester"
    field.cattle.io/projectId: "local:p-devteam"
spec:
  # K3s configuration
  rkeConfig:
    machinePools:
      - name: all-in-one
        quantity: 3   # 3 nodes for HA K3s
        etcdRole: true
        controlPlaneRole: true
        workerRole: true
        machineConfigRef:
          kind: HarvesterConfig
          name: k3s-small-node-config
    # K3s-specific install arguments
    machineGlobalConfig:
      # Disable traefik (will use nginx ingress instead)
      disable: "traefik"
      # Disable default storage class (will use Harvester CSI)
      disable-local-storage: true
      # Cluster DNS domain
      cluster-domain: "dev.cluster.local"
    # Upgrade configuration
    upgradeStrategy:
      controlPlaneConcurrency: "1"
      workerConcurrency: "1"
  kubernetesVersion: "v1.27.9+k3s1"
  networkConfig:
    plugin: flannel
```

```yaml
# k3s-small-node-config.yaml
# VM configuration for K3s nodes (smaller than RKE2 nodes)

apiVersion: rke-machine-config.cattle.io/v1
kind: HarvesterConfig
metadata:
  name: k3s-small-node-config
  namespace: fleet-default
spec:
  clusterName: local
  namespace: default
  # Use a lightweight image
  imageName: default/ubuntu-22-04-lts
  networkName: default/management
  # Smaller resources for dev cluster
  cpuCount: "2"
  memorySize: "4"  # GB
  diskSize: "30"   # GB
  diskStorageClassName: longhorn
  sshUser: ubuntu
  # Minimal cloud-init for K3s nodes
  userData: |
    #cloud-config
    packages:
      - qemu-guest-agent
    runcmd:
      - systemctl enable --now qemu-guest-agent
      # K3s prerequisites
      - modprobe br_netfilter
      - modprobe overlay
      - echo 'br_netfilter' >> /etc/modules-load.d/k3s.conf
      - echo 'overlay' >> /etc/modules-load.d/k3s.conf
      - sysctl --system
```

```bash
kubectl apply -f k3s-small-node-config.yaml
kubectl apply -f k3s-harvester-cluster.yaml

# Watch the K3s cluster provision
kubectl get cluster dev-k3s-cluster -n fleet-default -w
```

## Step 3: Manual K3s Setup on Harvester VMs

For more control, install K3s manually on Harvester VMs:

### Create VMs for K3s

```bash
# Create the K3s server VM
kubectl apply -f - <<EOF
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: k3s-server-01
  namespace: default
spec:
  running: true
  template:
    spec:
      domain:
        cpu:
          cores: 4
        resources:
          requests:
            memory: 8Gi
        machine:
          type: q35
        devices:
          disks:
            - name: rootdisk
              bootOrder: 1
              disk:
                bus: virtio
            - name: cloudinit
              disk:
                bus: virtio
          interfaces:
            - name: default
              masquerade: {}
      networks:
        - name: default
          pod: {}
      volumes:
        - name: rootdisk
          persistentVolumeClaim:
            claimName: k3s-server-01-root
        - name: cloudinit
          cloudInitNoCloud:
            userData: |
              #cloud-config
              hostname: k3s-server-01
              users:
                - name: ubuntu
                  sudo: ALL=(ALL) NOPASSWD:ALL
                  ssh_authorized_keys:
                    - ssh-ed25519 AAAAC3NzaC1... admin@host
              packages:
                - qemu-guest-agent
                - curl
              runcmd:
                - systemctl enable --now qemu-guest-agent
EOF
```

### Install K3s Server

```bash
# SSH into the K3s server VM
ssh ubuntu@<k3s-server-ip>

# Install K3s server
curl -sfL https://get.k3s.io | sh -s - \
    --cluster-init \
    --token="my-k3s-secret-token" \
    --tls-san="10.0.0.100" \
    --disable=traefik \
    --write-kubeconfig-mode=644

# Get the cluster token (needed for joining agents)
sudo cat /var/lib/rancher/k3s/server/node-token
```

### Install K3s Agent (Worker Nodes)

```bash
# On worker nodes, install K3s agent
curl -sfL https://get.k3s.io | K3S_URL=https://<server-ip>:6443 \
    K3S_TOKEN="my-k3s-secret-token" sh -
```

### Install K3s HA (3 Server Nodes)

```bash
# On server node 1 (init node)
curl -sfL https://get.k3s.io | sh -s - server \
    --cluster-init \
    --token="my-k3s-ha-token"

# On server nodes 2 and 3 (join the HA cluster)
curl -sfL https://get.k3s.io | sh -s - server \
    --server="https://<node1-ip>:6443" \
    --token="my-k3s-ha-token"
```

## Step 4: Configure Harvester CSI in K3s

Install the Harvester CSI driver in your K3s cluster for persistent storage:

```bash
# Download the Harvester kubeconfig
export KUBECONFIG=./harvester.kubeconfig

# Create a secret with Harvester connection details
kubectl create secret generic harvester-csi-controller-sa \
    --from-file=kubeconfig=./harvester.kubeconfig \
    -n kube-system

# Install the Harvester CSI driver (in the guest K3s cluster)
export KUBECONFIG=./k3s.kubeconfig

kubectl apply -f https://raw.githubusercontent.com/harvester/harvester-csi-driver/master/deploy/harvester-csi-driver.yaml
```

## Step 5: Access and Verify the K3s Cluster

```bash
# Get K3s kubeconfig (from Rancher or directly from the server VM)
# Via Rancher:
kubectl get secret dev-k3s-cluster-kubeconfig -n fleet-default \
    -o jsonpath='{.data.value}' | base64 -d > dev-k3s.kubeconfig

# Directly from K3s server VM:
ssh ubuntu@<k3s-server-ip> sudo cat /etc/rancher/k3s/k3s.yaml > dev-k3s.kubeconfig
# Update the server URL to the public IP

# Verify the K3s cluster
export KUBECONFIG=dev-k3s.kubeconfig
kubectl get nodes
kubectl get pods -A

# Deploy a test workload
kubectl run test-nginx --image=nginx:alpine --port=80
kubectl expose pod test-nginx --type=NodePort --port=80
kubectl get svc test-nginx
```

## Conclusion

K3s on Harvester provides an excellent platform for development, testing, and edge workloads that don't require the full weight of production RKE2. The lightweight nature of K3s means you can run many more isolated clusters on the same Harvester infrastructure compared to RKE2, making it perfect for giving development teams their own dedicated Kubernetes environments. Rancher's integration with Harvester makes provisioning and managing fleets of K3s clusters simple and consistent.
