# How to Create Elemental Seed Images

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Elemental, Kubernetes, Seed Images, Edge, ISO

Description: Step-by-step guide to creating Elemental seed images (ISOs and raw disk images) that embed registration configuration for automated bare metal provisioning.

## Introduction

Elemental seed images are bootable media (ISOs, USB images, or PXE configurations) that contain everything needed for a machine to register itself with Rancher and install the Elemental OS. They embed the registration URL, CA certificates, and initial cloud-config so that the provisioning process is fully automated - no manual intervention required after booting.

## Prerequisites

- Elemental Operator installed
- A MachineRegistration resource configured
- Docker or Podman
- `elemental` CLI or container image

## Types of Seed Images

| Type | Use Case |
|------|----------|
| ISO | Physical media, virtual machines, iDRAC/iLO virtual media |
| Raw disk image | USB drives, SD cards |
| PXE/iPXE | Network boot environments |

## Step 1: Gather Registration Configuration

```bash
# Export registration URL

REG_URL=$(kubectl get machineregistration my-nodes \
  -n fleet-default \
  -o jsonpath='{.status.registrationURL}')

# Export CA certificate
kubectl get secret tls-rancher-internal-ca \
  -n cattle-system \
  -o jsonpath='{.data.cacerts\.pem}' | base64 -d > /tmp/ca.pem

echo "Registration URL: $REG_URL"
```

## Step 2: Create the Seed Image Configuration

```yaml
# seed-image-config.yaml
cloud-config:
  # Initial user setup
  users:
    - name: root
      passwd: "$6$rounds=4096$salt$hashedpassword"

  # SSH configuration
  ssh_authorized_keys:
    - "ssh-rsa AAAAB3... admin@example.com"

  # Custom scripts run at first boot
  runcmd:
    - systemctl enable --now elemental-agent

elemental:
  registration:
    uri: "https://rancher.example.com/v1/elemental/registration/your-token"
    ca-cert: |
      -----BEGIN CERTIFICATE-----
      MIIDXTCCAkWgAwIBAgIJAMw...
      -----END CERTIFICATE-----
    no-smbios: false

  install:
    # Target device for OS installation
    device: /dev/sda
    # Reboot after successful install
    reboot: true
    # Don't power off (mutually exclusive with reboot)
    poweroff: false
    # Debug logging during install
    debug: false
    # Set hostname from system UUID
    system-uri: ""
```

## Step 3: Build an ISO Seed Image

```bash
# Build ISO using the elemental CLI container
docker run --privileged --rm \
  -v $(pwd):/workspace \
  -v /tmp/ca.pem:/ca.pem \
  registry.suse.com/rancher/elemental-toolkit/elemental-cli:latest \
  build-iso \
  --config /workspace/seed-image-config.yaml \
  --output /workspace/ \
  --name elemental-seed \
  registry.suse.com/rancher/sle-micro:latest

# Verify the ISO was created
ls -lh elemental-seed.iso
```

## Step 4: Build a Raw Disk Image (USB)

```bash
# Build raw disk image for USB deployment
docker run --privileged --rm \
  -v $(pwd):/workspace \
  registry.suse.com/rancher/elemental-toolkit/elemental-cli:latest \
  build-disk \
  --config /workspace/seed-image-config.yaml \
  --output /workspace/ \
  --name elemental-seed \
  registry.suse.com/rancher/sle-micro:latest

# Write to USB drive (replace /dev/sdX with your USB device)
dd if=elemental-seed.raw of=/dev/sdX bs=4M status=progress
sync
```

## Step 5: Customize the Seed Image

### Adding Custom Files

```dockerfile
# custom-seed.Dockerfile
FROM registry.suse.com/rancher/sle-micro:latest

# Install additional tools
RUN zypper --non-interactive install \
    vim \
    jq \
    && zypper clean -a

# Copy custom configuration
COPY my-custom-config.yaml /etc/elemental/config.d/
COPY custom-scripts/ /usr/local/bin/
```

```bash
# Build custom image first
docker build -t my-elemental-seed:latest -f custom-seed.Dockerfile .

# Then build ISO from custom image
docker run --privileged --rm \
  -v $(pwd):/workspace \
  registry.suse.com/rancher/elemental-toolkit/elemental-cli:latest \
  build-iso \
  --config /workspace/seed-image-config.yaml \
  --output /workspace/ \
  --name elemental-custom-seed \
  my-elemental-seed:latest
```

## Verifying the Seed Image

```bash
# Test the ISO in a VM (requires QEMU/KVM)
qemu-system-x86_64 \
  -m 4096 \
  -cdrom elemental-seed.iso \
  -boot d \
  -hda /tmp/test-disk.qcow2 \
  -nographic

# Or test with VirtualBox
VBoxManage createvm --name "elemental-test" --register
VBoxManage modifyvm "elemental-test" --memory 4096 --cpus 2
VBoxManage storagectl "elemental-test" --name "IDE" --add ide
VBoxManage storageattach "elemental-test" \
  --storagectl "IDE" --port 1 --device 0 \
  --type dvddrive --medium elemental-seed.iso
```

## Distributing Seed Images

```bash
# Upload to an S3-compatible bucket
aws s3 cp elemental-seed.iso s3://my-bucket/elemental-images/

# Or serve via HTTP
python3 -m http.server 8080 --directory $(pwd)
```

## Conclusion

Elemental seed images provide the zero-touch provisioning foundation for your edge and bare metal fleet. By embedding registration configuration directly into the boot media, you eliminate manual steps and enable machines to self-configure from the moment they boot. Combined with automated hardware provisioning tools like PXE, you can scale to hundreds of nodes with minimal operational overhead.
