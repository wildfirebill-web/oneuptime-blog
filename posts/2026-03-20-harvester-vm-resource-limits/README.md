# How to Configure VM Resource Limits in Harvester

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Harvester, Kubernetes, Virtualization, HCI, Resources, CPU, Memory

Description: Learn how to configure CPU, memory, and storage resource limits for virtual machines in Harvester to ensure fair resource allocation and prevent resource contention.

## Introduction

Resource management in Harvester VMs involves configuring CPU cores, memory limits, CPU pinning, and NUMA topology. Proper resource configuration ensures VMs get the resources they need while preventing any single VM from consuming resources that starve other workloads. Harvester inherits Kubernetes resource model principles, so understanding Kubernetes requests and limits is helpful.

Resource Configuration Concepts

| Setting | Description | Impact |
|---|---|---|
| CPU Cores | Number of virtual CPU cores | Determines vCPU scheduling |
| CPU Sockets/Threads | vCPU topology | Affects NUMA locality |
| Memory Requests | Guaranteed memory | Minimum reserved on node |
| Memory Limits | Maximum memory | Hard cap (OOM kill if exceeded) |
| CPU Pinning | Dedicated physical CPUs | Eliminates CPU contention |
| Hugepages | Large memory pages | Reduces TLB pressure |

## Step 1: Configure Basic CPU and Memory

```yaml
# vm-resource-config.yaml

apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: production-vm-01
  namespace: default
spec:
  running: true
  template:
    spec:
      domain:
        # CPU configuration
        cpu:
          # Number of virtual CPU cores
          cores: 8
          # Number of CPU sockets (usually 1)
          sockets: 1
          # Number of threads per core (use 2 for hyperthreading awareness)
          threads: 2
          # Total vCPUs = cores * sockets * threads = 8 * 1 * 2 = 16
        # Memory configuration
        resources:
          # Requests: guaranteed minimum memory for the VM
          requests:
            memory: 16Gi
            # CPU request: 8 cores guaranteed
            cpu: "8"
          # Limits: hard maximum (VM gets OOM killed if exceeded)
          limits:
            memory: 16Gi
            # CPU limit = CPU request for VMs (prevents overcommit)
            cpu: "8"
        machine:
          type: q35
        devices:
          disks:
            - name: rootdisk
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
            claimName: production-vm-01-root
```

## Step 2: Configure CPU Pinning for Low-Latency Workloads

CPU pinning dedicates physical CPU cores exclusively to a VM, eliminating CPU scheduling noise:

```yaml
# vm-cpu-pinning.yaml
# CPU pinning for latency-sensitive workloads (databases, real-time apps)

apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: latency-sensitive-vm
  namespace: default
spec:
  running: true
  template:
    spec:
      domain:
        cpu:
          cores: 4
          sockets: 1
          threads: 1
          # Enable CPU pinning - dedicates physical cores to this VM
          dedicatedCpuPlacement: true
          # NUMA-aware scheduling
          numa:
            guestMappingPassthrough: {}
        resources:
          # Must specify limits equal to requests for CPU pinning
          requests:
            memory: 8Gi
            cpu: "4"
          limits:
            memory: 8Gi
            cpu: "4"
        machine:
          type: q35
        devices:
          disks:
            - name: rootdisk
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
            claimName: latency-sensitive-vm-root
```

**Prerequisite for CPU pinning:**
```bash
# The node must have CPU management policy set to static
# Check on a node:
cat /var/lib/kubelet/config.yaml | grep cpuManager

# If not set, Harvester nodes need the CPU manager enabled
# This is usually configured at cluster creation time
```

## Step 3: Configure Hugepages for Memory Performance

Hugepages reduce TLB pressure and improve memory performance for large-memory VMs:

```yaml
# vm-hugepages.yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: database-vm-01
  namespace: default
spec:
  running: true
  template:
    spec:
      domain:
        cpu:
          cores: 8
          dedicatedCpuPlacement: true
        resources:
          requests:
            memory: 32Gi
            cpu: "8"
          limits:
            memory: 32Gi
            cpu: "8"
        # Configure hugepages
        memory:
          # Use 1GB hugepages
          hugepages:
            pageSize: 1Gi
          # Guest memory size (must match resources)
          guest: 32Gi
        machine:
          type: q35
        devices:
          disks:
            - name: rootdisk
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
            claimName: database-vm-01-root
```

```bash
# Check hugepage availability on nodes
kubectl get node harvester-node-01 \
    -o jsonpath='{.status.allocatable}' | jq .

# Pre-allocate hugepages on nodes (requires node restart)
# Add to /etc/sysctl.conf:
# vm.nr_hugepages = 32  (32 x 1GB hugepages)
```

## Step 4: Set Resource Limits via the UI

When creating a VM in the Harvester UI:

1. **CPU**: Set the number of cores in the **CPU** field
2. **Memory**: Set the memory in Gi or Mi in the **Memory** field
3. Click **Advanced** for CPU pinning and NUMA options

## Step 5: Check Resource Utilization

```bash
# View resource requests/limits for all VMs
kubectl get vmi -n default \
    -o custom-columns=\
'NAME:.metadata.name,CPU:.spec.domain.resources.requests.cpu,MEM:.spec.domain.resources.requests.memory'

# Check actual resource consumption of the virt-launcher pod
kubectl top pods -n default \
    -l vm.kubevirt.io/name=production-vm-01

# View node capacity vs allocated resources
kubectl describe node harvester-node-01 | grep -A 10 "Allocated resources"

# Example output:
# Allocated resources:
#   CPU Requests: 24 (60%)
#   Memory Requests: 64Gi (80%)
```

## Step 6: Implement Resource Quotas

Use Kubernetes ResourceQuotas to limit total VM resources per namespace:

```yaml
# vm-resource-quota.yaml
# Limit total VM resources in the 'development' namespace

apiVersion: v1
kind: ResourceQuota
metadata:
  name: vm-resource-quota
  namespace: development
spec:
  hard:
    # Maximum total vCPUs across all VMs in this namespace
    requests.cpu: "32"
    limits.cpu: "32"
    # Maximum total memory
    requests.memory: 128Gi
    limits.memory: 128Gi
    # Maximum number of PVCs (VM disks)
    persistentvolumeclaims: "20"
    # Maximum total PVC storage
    requests.storage: 2Ti
```

```bash
kubectl apply -f vm-resource-quota.yaml

# Check quota usage
kubectl describe resourcequota vm-resource-quota -n development
```

## Step 7: VM Vertical Scaling

To change resources on an existing VM:

```bash
# Stop the VM first (resource changes require restart for most settings)
kubectl patch vm production-vm-01 -n default \
    --type merge \
    -p '{"spec":{"running":false}}'

# Wait for VM to stop
kubectl wait vmi/production-vm-01 -n default \
    --for delete --timeout=120s

# Update CPU and memory
kubectl patch vm production-vm-01 -n default \
    --type merge \
    -p '{
        "spec": {
            "template": {
                "spec": {
                    "domain": {
                        "cpu": {"cores": 16},
                        "resources": {
                            "requests": {"memory": "32Gi", "cpu": "16"},
                            "limits": {"memory": "32Gi", "cpu": "16"}
                        }
                    }
                }
            }
        }
    }'

# Start the VM with new resources
kubectl patch vm production-vm-01 -n default \
    --type merge \
    -p '{"spec":{"running":true}}'
```

## Conclusion

Proper resource configuration in Harvester ensures VMs get the performance they need while maintaining cluster-wide stability. For most workloads, setting appropriate CPU and memory requests is sufficient. For high-performance databases and latency-sensitive applications, CPU pinning and hugepages provide the dedicated, predictable resources needed. ResourceQuotas help prevent runaway resource consumption in shared environments. Always monitor actual resource utilization and adjust configurations based on real workload patterns rather than guesses.
