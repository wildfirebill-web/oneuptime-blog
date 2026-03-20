# How to Set Up Nested Virtualization in Harvester

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Harvester, Kubernetes, Virtualization, HCI, Nested Virtualization, KVM

Description: Learn how to enable and configure nested virtualization in Harvester VMs, allowing virtual machines to run hypervisors and nested VMs.

## Introduction

Nested virtualization allows a virtual machine running on Harvester to itself run virtual machines or containers using hardware virtualization. This is useful for creating virtualization labs, testing hypervisor software, running Kubernetes with VM support (like KubeVirt in a VM), or validating VM configurations before applying them to production. Harvester supports nested virtualization by exposing the host CPU's virtualization extensions to guest VMs.

## Use Cases

- **Virtualization labs**: Test KVM, VMware, or Hyper-V configurations
- **CI/CD testing**: Test VM creation scripts in isolated environments
- **Nested Kubernetes**: Run Harvester or KubeVirt inside a Harvester VM
- **Training environments**: Create sandboxed environments for learning

## Prerequisites

- Harvester cluster with hosts that have Intel VT-x or AMD-V enabled
- The host node must expose virtualization extensions to guests
- Nested virtualization support in the host kernel (Intel KVM: `kvm_intel.nested=1`)

## Step 1: Enable Nested Virtualization on Harvester Nodes

```bash
# SSH into each Harvester node
ssh rancher@192.168.1.11

# Check if nested virtualization is already enabled
cat /sys/module/kvm_intel/parameters/nested  # For Intel
cat /sys/module/kvm_amd/parameters/nested    # For AMD

# If the output is 'Y' or '1', it's already enabled

# Enable nested virtualization for Intel KVM
echo "options kvm_intel nested=1" | sudo tee /etc/modprobe.d/kvm-intel.conf

# For AMD:
echo "options kvm_amd nested=1" | sudo tee /etc/modprobe.d/kvm-amd.conf

# Reload the KVM module (requires downtime if VMs are running)
sudo modprobe -r kvm_intel && sudo modprobe kvm_intel nested=1
# Or just reboot the node
sudo reboot

# Verify after reboot
cat /sys/module/kvm_intel/parameters/nested
# Expected output: Y
```

## Step 2: Create a VM with Nested Virtualization

To enable nested virtualization in a VM, you need to pass through the CPU virtualization features:

```yaml
# vm-nested-virt.yaml
# VM configured for nested virtualization

apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: virt-lab-vm
  namespace: default
  labels:
    purpose: nested-virtualization
spec:
  running: true
  template:
    spec:
      domain:
        # CPU configuration for nested virtualization
        cpu:
          cores: 8
          sockets: 1
          threads: 2
          # CRITICAL: Pass host CPU features to enable nested virt
          model: host-passthrough
          # Or use host-model for better compatibility
          # model: host-model
          # Explicitly enable virtualization features
          features:
            - name: vmx      # Intel VT-x
              policy: require
            - name: svm      # AMD-V (include both, KubeVirt will use the right one)
              policy: optional
        resources:
          requests:
            memory: 32Gi
          limits:
            memory: 32Gi
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
              model: virtio
              masquerade: {}
      networks:
        - name: default
          pod: {}
      volumes:
        - name: rootdisk
          persistentVolumeClaim:
            claimName: virt-lab-vm-root
        - name: cloudinit
          cloudInitNoCloud:
            userData: |
              #cloud-config
              users:
                - name: ubuntu
                  sudo: ALL=(ALL) NOPASSWD:ALL
                  ssh_authorized_keys:
                    - ssh-ed25519 AAAAC3NzaC1... admin@host
              packages:
                - qemu-guest-agent
                - qemu-kvm
                - libvirt-daemon-system
                - libvirt-clients
                - bridge-utils
                - virtinst
              runcmd:
                - systemctl enable --now qemu-guest-agent
                - systemctl enable --now libvirtd
                - usermod -aG libvirt ubuntu
```

## Step 3: Verify Nested Virtualization Inside the VM

```bash
# SSH into the nested VM
ssh ubuntu@<vm-ip>

# Check if virtualization extensions are visible
egrep -c '(vmx|svm)' /proc/cpuinfo
# Expected: > 0 means virtualization extensions are available

# Check KVM support
kvm-ok
# Expected: "KVM acceleration can be used"

# Check with virt-host-validate
virt-host-validate

# Verify hardware virtualization
ls /dev/kvm
# Expected: /dev/kvm exists
```

## Step 4: Create a Nested VM with KVM

```bash
# Inside the nested VM, create a simple test VM
# First, download a cloud image to test with
wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img -O ubuntu22.img

# Create a VM using virt-install
virt-install \
    --name nested-vm-01 \
    --ram 2048 \
    --vcpus 2 \
    --disk path=ubuntu22.img,format=qcow2 \
    --import \
    --os-variant ubuntu22.04 \
    --network network=default \
    --graphics none \
    --serial pty \
    --console pty \
    --noautoconsole

# Check the nested VM is running
virsh list --all

# Connect to the nested VM console
virsh console nested-vm-01
```

## Step 5: Set Up a Nested Kubernetes Cluster

A powerful use case: run a K3s cluster inside a Harvester VM:

```yaml
# vm-for-nested-k8s.yaml
# VM for running a nested Kubernetes cluster

apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: nested-k8s-vm
  namespace: default
spec:
  running: true
  template:
    spec:
      domain:
        cpu:
          cores: 8
          model: host-passthrough
        resources:
          requests:
            memory: 16Gi
          limits:
            memory: 16Gi
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
            claimName: nested-k8s-vm-root
        - name: cloudinit
          cloudInitNoCloud:
            userData: |
              #cloud-config
              packages:
                - qemu-guest-agent
                - qemu-kvm
                - curl
              runcmd:
                - systemctl enable --now qemu-guest-agent
                # Install K3s
                - curl -sfL https://get.k3s.io | sh -
                # Wait for K3s to be ready
                - sleep 30
                - kubectl wait node --all --for=condition=Ready --timeout=120s
                # Show cluster info
                - kubectl get nodes
```

## Step 6: Performance Considerations

Nested virtualization has overhead at each layer:

```
Layer 1: Physical hardware
Layer 2: Harvester KVM (5-15% overhead)
Layer 3: Guest VM KVM (additional 5-15% overhead)
```

```bash
# Check nested VM performance vs bare metal
# CPU benchmark inside nested VM
sysbench cpu --cpu-max-prime=20000 run

# Compare with:
# - Bare metal: ~100% baseline
# - Harvester VM (no nesting): ~90-95% of bare metal
# - Nested VM: ~75-85% of bare metal

# Memory performance test
sysbench memory --memory-block-size=1M --memory-total-size=10G run
```

**Optimization tips:**
```yaml
# Use hugepages in the VM for better memory performance
domain:
  memory:
    hugepages:
      pageSize: 2Mi
  resources:
    requests:
      memory: 16Gi

# Use CPU pinning to avoid scheduler interference
cpu:
  cores: 8
  dedicatedCpuPlacement: true
```

## Conclusion

Nested virtualization in Harvester enables powerful use cases like virtualization labs, CI testing, and Kubernetes-on-Kubernetes architectures. The key configuration is exposing the host CPU's VMX/SVM features to the guest VM using `model: host-passthrough`. While there is a performance overhead compared to running workloads directly on the bare metal or in a flat VM, the isolation and flexibility benefits make nested virtualization valuable for development, testing, and training environments. For production workloads, prefer running VMs directly on Harvester rather than inside nested VMs.
