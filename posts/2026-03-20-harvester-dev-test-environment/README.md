# How to Set Up Harvester for Dev/Test Environments

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Harvester, Dev/Test, Virtual Machine, Kubernetes, Development Environments, SUSE Rancher, HCI

Description: Learn how to set up a Harvester cluster for development and testing environments including self-service VM provisioning, resource quotas, network isolation, and template-based VM creation.

---

Harvester is well-suited for dev/test environments because it combines virtual machine management and Kubernetes in a single platform. Development teams can provision VMs or Kubernetes clusters on-demand without managing separate virtualization and container infrastructure.

---

## Dev/Test Architecture

```text
┌──────────────────────────────────────────────┐
│              Harvester Cluster               │
│                                              │
│  ┌──────────────┐  ┌───────────────────────┐ │
│  │  Dev VMs     │  │    Test K8s Clusters  │ │
│  │              │  │   (RKE2/K3s on VMs)   │ │
│  │  dev-vm-1    │  │   test-cluster-1      │ │
│  │  dev-vm-2    │  │   test-cluster-2      │ │
│  └──────────────┘  └───────────────────────┘ │
│                                              │
│  ┌──────────────────────────────────────┐    │
│  │  Development Network (isolated VLAN) │    │
│  └──────────────────────────────────────┘    │
└──────────────────────────────────────────────┘
```

---

## Step 1: Create a Dedicated Network for Dev/Test

```yaml
# dev-network.yaml

apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: dev-network
  namespace: default
  annotations:
    network.harvesterhci.io/route: |
      {"mode":"auto","serverIPAddr":"","cidr":"","gateway":""}
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "name": "dev-network",
      "type": "bridge",
      "bridge": "dev-br",
      "ipam": {
        "type": "whereabouts",
        "range": "192.168.100.0/24",
        "gateway": "192.168.100.1"
      }
    }
```

```bash
kubectl apply -f dev-network.yaml
```

---

## Step 2: Create VM Templates for Common Development Images

In the Harvester UI, create VM templates that developers can reuse:

1. Go to **Virtual Machines** → **VM Templates**
2. Create templates for:
   - Ubuntu 22.04 LTS (developer workstation)
   - CentOS Stream 9 (RHEL-compatible testing)
   - Windows Server 2022 (Windows development)

```yaml
# Example VM template via kubectl
apiVersion: harvesterhci.io/v1beta1
kind: VirtualMachineTemplate
metadata:
  name: ubuntu-dev
  namespace: default
spec:
  versionName: ubuntu-22.04-dev-v1
```

---

## Step 3: Configure Resource Quotas per Namespace

```yaml
# dev-namespace-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: dev-team
spec:
  hard:
    # Limit VM CPU and memory
    requests.cpu: "32"
    limits.cpu: "64"
    requests.memory: 64Gi
    limits.memory: 128Gi
    # Limit number of VMs
    count/virtualmachines.kubevirt.io: "20"
    # Limit storage
    requests.storage: 2Ti
```

---

## Step 4: Create a Self-Service VM Provisioning Script

```bash
#!/bin/bash
# create-dev-vm.sh
# Usage: ./create-dev-vm.sh <vm-name> <developer-name>

VM_NAME=$1
DEVELOPER=$2
NAMESPACE="dev-${DEVELOPER}"

# Create namespace if it doesn't exist
kubectl create namespace $NAMESPACE 2>/dev/null

# Create the VM
kubectl apply -f - <<EOF
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: $VM_NAME
  namespace: $NAMESPACE
  labels:
    developer: $DEVELOPER
    environment: dev
spec:
  running: true
  template:
    metadata:
      labels:
        kubevirt.io/vm: $VM_NAME
    spec:
      domain:
        cpu:
          cores: 4
        memory:
          guest: 8Gi
        devices:
          disks:
            - name: root-disk
              disk:
                bus: virtio
          interfaces:
            - name: dev-net
              bridge: {}
      networks:
        - name: dev-net
          multus:
            networkName: dev-network
      volumes:
        - name: root-disk
          dataVolume:
            name: ${VM_NAME}-root
---
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: ${VM_NAME}-root
  namespace: $NAMESPACE
spec:
  source:
    registry:
      url: docker://registry.example.com/images/ubuntu-22.04:latest
  pvc:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 50Gi
    storageClassName: longhorn
EOF

echo "VM $VM_NAME created in namespace $NAMESPACE"
echo "VNC access: kubectl virtctl vnc $VM_NAME -n $NAMESPACE"
```

---

## Step 5: Automate VM Cleanup

```yaml
# vm-cleanup-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cleanup-old-dev-vms
  namespace: default
spec:
  schedule: "0 22 * * 5"    # Every Friday at 10 PM
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: vm-cleanup-sa
          containers:
            - name: cleanup
              image: bitnami/kubectl:latest
              command:
                - sh
                - -c
                - |
                  # Delete VMs older than 7 days with the dev label
                  kubectl get vm -A -l environment=dev -o json | \
                  jq -r '.items[] | select(
                    (now - (.metadata.creationTimestamp | fromdateiso8601)) > 604800
                  ) | "\(.metadata.namespace)/\(.metadata.name)"' | \
                  while read vm; do
                    NS=$(echo $vm | cut -d/ -f1)
                    NAME=$(echo $vm | cut -d/ -f2)
                    kubectl delete vm $NAME -n $NS
                  done
          restartPolicy: OnFailure
```

---

## Step 6: Set Up Test Kubernetes Clusters on Harvester

Use Rancher to provision ephemeral K3s clusters on Harvester VMs for integration testing:

```bash
# Create a test cluster via Rancher API
# Configure: Harvester Infrastructure Provider → K3s → 1 server + 2 agents
# Enable: Auto-cleanup after test completion

# Or use the Rancher UI:
# Cluster Management → Create → Harvester → Select K3s
```

---

## Best Practices

- Create isolated VLANs per development team so test workloads cannot interfere with each other or with production networks.
- Implement automated VM cleanup for idle or old VMs - storage costs accumulate quickly when developers forget to delete test VMs.
- Use VM templates for common OS images and pre-install developer tools in the base images - this reduces VM provisioning time from minutes to seconds.
