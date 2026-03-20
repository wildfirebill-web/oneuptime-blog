# How to Set Up Elemental with USB Boot

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Elemental, USB Boot, Edge, Kubernetes, Provisioning

Description: Create bootable USB drives with Elemental registration images for provisioning bare metal nodes without network boot infrastructure.

## Introduction

USB boot is the simplest provisioning method for Elemental, requiring no special network infrastructure. You create a bootable USB drive containing the Elemental OS image and registration configuration, insert it into a machine, boot, and the machine provisions itself automatically. This approach is ideal for remote sites, retail locations, or any environment without PXE infrastructure.

## Prerequisites

- A USB drive (8 GB or larger)
- Elemental CLI or Docker
- Registration URL from your MachineRegistration
- `dd` utility (Linux/macOS) or Rufus (Windows)

## Step 1: Gather Registration Information

```bash
# Get registration URL

REG_URL=$(kubectl get machineregistration my-nodes \
  -n fleet-default \
  -o jsonpath='{.status.registrationURL}')

# Get CA certificate
kubectl get secret tls-rancher-internal-ca \
  -n cattle-system \
  -o jsonpath='{.data.cacerts\.pem}' | base64 -d > /tmp/ca.pem

echo "Registration URL: $REG_URL"
```

## Step 2: Create Registration Configuration

```yaml
# usb-registration-config.yaml
cloud-config:
  users:
    - name: root
      passwd: "$6$rounds=4096$salt$hashedpassword"
  ssh_authorized_keys:
    - "ssh-rsa AAAAB3... admin@example.com"

elemental:
  registration:
    uri: "https://rancher.example.com/v1/elemental/registration/your-token"
    ca-cert: |
      -----BEGIN CERTIFICATE-----
      MIIDXTCCAkWgAwIBAgIJ...
      -----END CERTIFICATE-----

  install:
    device: /dev/sda
    reboot: true
    debug: false
```

## Step 3: Build the USB Image

### Build ISO for USB

```bash
# Build ISO with embedded registration config
docker run --privileged --rm \
  -v $(pwd):/workspace \
  registry.suse.com/rancher/elemental-toolkit/elemental-cli:latest \
  build-iso \
  --config /workspace/usb-registration-config.yaml \
  --output /workspace/ \
  --name elemental-usb \
  registry.suse.com/rancher/sle-micro:latest

ls -lh elemental-usb.iso
```

### Build Raw Disk Image (Better for USB)

```bash
# Build raw disk image optimized for USB
docker run --privileged --rm \
  -v $(pwd):/workspace \
  registry.suse.com/rancher/elemental-toolkit/elemental-cli:latest \
  build-disk \
  --config /workspace/usb-registration-config.yaml \
  --output /workspace/ \
  --name elemental-usb \
  registry.suse.com/rancher/sle-micro:latest
```

## Step 4: Write to USB Drive

### On Linux

```bash
# Find the USB device
lsblk

# Write ISO to USB (CAUTION: verify the device path!)
sudo dd if=elemental-usb.iso of=/dev/sdX bs=4M status=progress oflag=sync

# Or write raw disk image
sudo dd if=elemental-usb.raw of=/dev/sdX bs=4M status=progress oflag=sync

# Sync and eject
sync
sudo eject /dev/sdX
```

### On macOS

```bash
# Find USB device
diskutil list

# Unmount (but don't eject)
diskutil unmountDisk /dev/diskX

# Write image (note: rdisk for faster writes)
sudo dd if=elemental-usb.iso of=/dev/rdiskX bs=4m

# Eject
diskutil eject /dev/diskX
```

### On Windows (Using Rufus or dd)

```powershell
# Using dd for Windows
.\dd.exe if=elemental-usb.iso of=\\.\PhysicalDriveX bs=4M

# Or use Rufus (GUI tool):
# 1. Open Rufus
# 2. Select USB drive
# 3. Select elemental-usb.iso
# 4. Use DD Image mode
# 5. Click START
```

## Step 5: Boot the Machine

1. Insert the USB drive into the target machine
2. Enter BIOS/UEFI settings (usually F2, F10, F12, or Del)
3. Set USB as the first boot device
4. Save and reboot
5. The machine boots Elemental, installs to /dev/sda, and reboots
6. After reboot, the machine registers with Rancher

## Step 6: Monitor Registration

```bash
# Watch for the machine to appear in inventory
kubectl get machineinventory -n fleet-default --watch

# Once registered, verify the machine
kubectl describe machineinventory -n fleet-default <machine-name>
```

## Creating Multiple USB Drives

For large deployments, create multiple identical USB drives:

```bash
# Write to multiple USB drives in parallel
for device in /dev/sdb /dev/sdc /dev/sdd; do
  sudo dd if=elemental-usb.iso of=$device bs=4M status=progress &
done

# Wait for all to complete
wait
echo "All USB drives written"
```

## Conclusion

USB boot provides a simple, reliable provisioning method for Elemental nodes that doesn't require any network boot infrastructure. By creating standardized USB images with embedded registration configuration, field technicians can provision nodes at remote locations with just a USB drive and no additional tooling. Once the USB is inserted and the machine boots, the entire provisioning process is automated.
