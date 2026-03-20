# How to Attach Volumes to VMs in Harvester

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Harvester, Kubernetes, Virtualization, HCI, Storage, Volumes, KubeVirt

Description: A guide to attaching existing storage volumes to virtual machines in Harvester, including hot-plug support and multi-disk configurations.

## Introduction

Attaching volumes to VMs in Harvester allows you to add persistent data disks to running or stopped virtual machines. Harvester supports both cold attachment (VM must be stopped) and hot-plug (attach to a running VM) for data disks. This is useful for adding database storage, shared data volumes, or migrating data between VMs.

## Prerequisites

- A running Harvester cluster
- An existing VM (running or stopped)
- A PVC (Persistent Volume Claim) that is not currently attached to another VM

## Method 1: Attach a Volume via the UI

### To a Stopped VM

1. Navigate to **Virtual Machines**
2. Click on the VM name to open its details
3. Click **Edit**
4. Go to the **Volumes** tab
5. Click **Add Volume**
6. Select **Use Existing Volume**
7. Choose the PVC from the dropdown
8. Set the bus type (VirtIO recommended for Linux, SATA for Windows)
9. Click **Save**

### Hot-Plug to a Running VM

1. Navigate to **Virtual Machines**
2. Click on the running VM
3. Click the **⋮** menu → **Add Volume**
4. Select the existing PVC
5. Click **Add** — the disk appears in the VM within seconds

## Method 2: Attach via kubectl (Cold Attach)

Edit the VM specification to add a new disk:

```yaml
# The VM spec needs both a disk entry and a volume entry

# First, check the current VM spec
kubectl get vm my-database-vm -n default -o yaml

# Apply a patch to add a new data disk
kubectl patch vm my-database-vm -n default --type json \
-p '[
  {
    "op": "add",
    "path": "/spec/template/spec/domain/devices/disks/-",
    "value": {
      "name": "datavolume1",
      "disk": {
        "bus": "virtio"
      }
    }
  },
  {
    "op": "add",
    "path": "/spec/template/spec/volumes/-",
    "value": {
      "name": "datavolume1",
      "persistentVolumeClaim": {
        "claimName": "database-data-500gb"
      }
    }
  }
]'
```

For a VM that hasn't been created yet, include volumes in the initial spec:

```yaml
# vm-with-multiple-disks.yaml
# VM with root disk + two data disks

apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: database-server-01
  namespace: default
spec:
  running: true
  template:
    spec:
      domain:
        cpu:
          cores: 8
        resources:
          requests:
            memory: 32Gi
        machine:
          type: q35
        devices:
          disks:
            # Boot disk - OS
            - name: rootdisk
              bootOrder: 1
              disk:
                bus: virtio
            # Data disk 1 - Database files
            - name: dbdata
              disk:
                bus: virtio
            # Data disk 2 - Database logs
            - name: dblogs
              disk:
                bus: virtio
            # Cloud-init disk
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
        # Boot volume (from image)
        - name: rootdisk
          persistentVolumeClaim:
            claimName: database-server-01-root
        # Data volumes
        - name: dbdata
          persistentVolumeClaim:
            claimName: database-data-500gb
        - name: dblogs
          persistentVolumeClaim:
            claimName: database-logs-100gb
        - name: cloudinit
          cloudInitNoCloud:
            userData: |
              #cloud-config
              # Format and mount data disks on first boot
              runcmd:
                # Format /dev/vdb for database data
                - mkfs.xfs /dev/vdb
                - mkdir -p /var/lib/postgresql/data
                - echo '/dev/vdb /var/lib/postgresql/data xfs defaults 0 2' >> /etc/fstab
                - mount -a
                # Format /dev/vdc for database logs
                - mkfs.xfs /dev/vdc
                - mkdir -p /var/log/postgresql
                - echo '/dev/vdc /var/log/postgresql xfs defaults 0 2' >> /etc/fstab
                - mount -a
```

## Method 3: Hot-Plug with virtctl

The `virtctl` tool supports hot-plugging volumes to running VMs:

```bash
# Install virtctl (if not already installed)
VERSION=$(kubectl get kubevirt -n harvester-system -o jsonpath='{.items[0].status.observedKubeVirtVersion}')
curl -L -o /usr/local/bin/virtctl \
    https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/virtctl-${VERSION}-linux-amd64
chmod +x /usr/local/bin/virtctl

# Hot-plug a volume to a running VM
virtctl addvolume my-database-vm \
    --volume-name=extra-storage \
    --persist \
    --serial=ext1 \
    -n default

# The --persist flag makes the attachment survive VM restarts
# Without --persist, the volume is detached on next VM restart
```

```bash
# Verify the volume was hot-plugged successfully
kubectl get vmi my-database-vm -n default \
    -o jsonpath='{.status.volumeStatus}' | jq .

# Inside the VM, the new disk should appear immediately
# Check for the new device
lsblk
```

## Step: Format and Mount Volumes Inside the VM

After attaching a new empty volume, format and mount it inside the VM:

```bash
# Access the VM console or SSH into it

# List block devices - new disk will appear as /dev/vdb, /dev/vdc, etc.
lsblk

# Format the new disk (WARNING: this erases all data on the disk)
sudo mkfs.ext4 -L data-disk /dev/vdb

# Create a mount point
sudo mkdir -p /mnt/data

# Mount temporarily
sudo mount /dev/vdb /mnt/data

# Add to fstab for persistent mounting
echo 'LABEL=data-disk /mnt/data ext4 defaults 0 2' | sudo tee -a /etc/fstab

# Verify fstab entry works
sudo umount /mnt/data
sudo mount -a
df -h /mnt/data
```

## Detaching Volumes

### Via virtctl (Hot Unplug)

```bash
# Hot unplug a volume from a running VM
virtctl removevolume my-database-vm \
    --volume-name=extra-storage \
    --persist \
    -n default
```

### Via kubectl (Cold Detach)

```bash
# Remove the disk from the VM spec (requires VM restart)
kubectl patch vm my-database-vm -n default --type json \
-p '[
  {
    "op": "remove",
    "path": "/spec/template/spec/domain/devices/disks/2"
  },
  {
    "op": "remove",
    "path": "/spec/template/spec/volumes/2"
  }
]'
```

## Troubleshooting Volume Attachment Issues

```bash
# Check VM events for attachment errors
kubectl get events -n default \
    --field-selector involvedObject.name=my-database-vm \
    --sort-by='.lastTimestamp'

# Check if PVC is already bound to another VM
kubectl get pvc database-data-500gb -n default \
    -o jsonpath='{.spec.volumeName}'

# Verify the PVC is in Bound state
kubectl get pvc -n default | grep database-data-500gb

# Check Longhorn volume is healthy
kubectl get volumes.longhorn.io -n longhorn-system | grep database-data
```

## Conclusion

Attaching volumes to VMs in Harvester is a flexible operation that supports both pre-planned multi-disk configurations and on-the-fly storage expansion via hot-plug. By separating application data from the OS disk, you make VMs more maintainable — you can recreate or upgrade the OS disk without touching data volumes. Hot-plug support means storage capacity can be increased without scheduling downtime, which is valuable for production database and application servers.
