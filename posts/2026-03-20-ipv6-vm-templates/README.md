# How to Configure IPv6 for Virtual Machine Templates

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, VM Templates, Cloud-init, Packer, VMware, KVM, Automation

Description: Create virtual machine templates with proper IPv6 network configuration using cloud-init, Packer, and platform-specific tools, ensuring new VMs receive IPv6 addresses on first boot.

## Introduction

VM templates are golden images used to create new virtual machines. IPv6 configuration in templates requires care: you should not embed static IPv6 addresses in templates (they'd conflict when multiple VMs are created), and SLAAC or DHCPv6 must be configured to activate on first boot. Cloud-init is the standard tool for per-instance IPv6 configuration, while Packer automates the template creation process.

## Cloud-init for IPv6 in VM Templates

```yaml
# user-data: cloud-init network configuration for IPv6

#cloud-config
network:
  version: 2
  ethernets:
    eth0:
      # Enable both DHCPv4 and DHCPv6/SLAAC
      dhcp4: true
      dhcp6: true

      # Or use SLAAC only (no DHCPv6):
      # dhcp4: true
      # ipv6-privacy: true    # Enable privacy extensions
      # accept-ra: true       # Accept Router Advertisements

      # Or fully static (parameterized):
      # Use cloud-init variables from metadata:
      addresses:
        - "{{ v4_address }}/24"     # from instance metadata
        - "{{ v6_address }}/64"     # from IPAM via metadata
      gateway4: "{{ v4_gateway }}"
      gateway6: "{{ v6_gateway }}"
      nameservers:
        addresses:
          - "2001:db8::53"
          - "8.8.8.8"
```

## KVM Template with cloud-init IPv6

```bash
# Create a base image with cloud-init ready for IPv6

# Download a cloud image
wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img

# Customize the image (remove machine-id so each VM gets unique ID)
virt-customize -a jammy-server-cloudimg-amd64.img \
    --run-command "truncate -s 0 /etc/machine-id" \
    --run-command "rm -f /var/lib/dbus/machine-id" \
    --run-command "ln -s /etc/machine-id /var/lib/dbus/machine-id"

# Create a cloud-init ISO for template testing
cat > user-data.yaml << 'EOF'
#cloud-config
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: true
      dhcp6: true
EOF

cat > meta-data.yaml << 'EOF'
instance-id: template-test
local-hostname: template-vm
EOF

cloud-localds seed.img user-data.yaml meta-data.yaml

# Test the template
qemu-system-x86_64 \
    -drive file=jammy-server-cloudimg-amd64.img,format=qcow2 \
    -drive file=seed.img,format=raw \
    -netdev bridge,id=net0,br=br0 \
    -device virtio-net-pci,netdev=net0 \
    -m 2048 -nographic
```

## VMware Template with IPv6 (vSphere customization)

```
# VMware vSphere guest customization spec for IPv6

1. vSphere Client → Policies and Profiles → VM Customization Specifications
2. Create New Specification
3. Under "Network Interface Settings":
   - Type: Use DHCP for IPv4
   - IPv6: Enable DHCPv6 or SLAAC
   - Or: Use fixed IPv6 (populated from template at clone time)

4. Use vSphere API to clone with IPv6 customization:
```

```python
#!/usr/bin/env python3
# vsphere_ipv6_template_clone.py

from pyVmomi import vim
from pyVim.connect import SmartConnect

def clone_with_ipv6(si, template_name: str, vm_name: str, ipv6_addr: str):
    """Clone a VMware template with IPv6 configuration."""

    # Get template
    template = get_vm_by_name(si, template_name)

    # Create customization spec
    ip_settings = vim.vm.customization.IPSettings()
    ip_settings.ip = vim.vm.customization.DhcpIpGenerator()

    # IPv6 settings
    ipv6_settings = vim.vm.customization.IPSettings.IpV6AddressSpec()
    ipv6_addr_obj = vim.vm.customization.FixedIpV6()
    ipv6_addr_obj.ipAddress = ipv6_addr
    ipv6_addr_obj.subnetMask = 64
    ipv6_settings.ip = [ipv6_addr_obj]
    ipv6_settings.gateway = "2001:db8::1"
    ip_settings.ipV6Spec = ipv6_settings

    nic_map = vim.vm.customization.AdapterMapping()
    nic_map.adapter = ip_settings

    spec = vim.vm.customization.Specification()
    spec.nicSettingMap = [nic_map]
    spec.identity = vim.vm.customization.LinuxPrep(
        hostName=vim.vm.customization.FixedName(name=vm_name),
        domain="example.com"
    )

    # Clone
    clone_spec = vim.vm.CloneSpec(
        customization=spec,
        powerOn=True,
    )
    task = template.Clone(name=vm_name, folder=template.parent, spec=clone_spec)
    return task
```

## Packer Template for IPv6-Ready Golden Image

```hcl
# packer/ubuntu-ipv6-template.pkr.hcl

packer {
  required_plugins {
    qemu = {
      version = ">= 1.0.0"
      source  = "github.com/hashicorp/qemu"
    }
  }
}

source "qemu" "ubuntu-ipv6" {
  iso_url          = "https://releases.ubuntu.com/22.04/ubuntu-22.04.4-live-server-amd64.iso"
  iso_checksum     = "sha256:..."
  disk_size        = "10240"
  memory           = 2048
  output_directory = "output"
  vm_name          = "ubuntu-ipv6-template.qcow2"

  # Use bridged networking with IPv6 during build
  net_device    = "virtio-net"
  disk_interface = "virtio"

  boot_command = [
    "<esc><wait>",
    "linux /casper/vmlinuz ip=dhcp ipv6.disable=0 autoinstall ds=nocloud-net;s=http://{{ .HTTPIP }}:{{ .HTTPPort }}/",
    "<enter>",
    "initrd /casper/initrd<enter>",
    "boot<enter>"
  ]

  http_directory = "http"
}

build {
  sources = ["source.qemu.ubuntu-ipv6"]

  # Clear instance-specific state
  provisioner "shell" {
    inline = [
      "sudo truncate -s 0 /etc/machine-id",
      "sudo rm -f /var/lib/dbus/machine-id",
      "sudo cloud-init clean --logs",
    ]
  }

  # Ensure cloud-init network config for IPv6 is in place
  provisioner "file" {
    source      = "files/99-ipv6-dhcp.yaml"
    destination = "/tmp/99-ipv6-dhcp.yaml"
  }

  provisioner "shell" {
    inline = [
      "sudo install -m 644 /tmp/99-ipv6-dhcp.yaml /etc/cloud/cloud.cfg.d/",
    ]
  }
}
```

## Template Pre-boot Checklist for IPv6

```bash
# Before converting a VM to a template, verify:

# 1. Machine ID is cleared (prevents duplicate DHCPv6 DUIDs)
cat /etc/machine-id
# Should be empty or "uninitialized"

# 2. No static IPv6 address is configured
grep -r "inet6" /etc/netplan/ /etc/network/interfaces 2>/dev/null
# Should show dhcp6:true or accept-ra:true, NOT addresses: - 2001:...

# 3. cloud-init is clean
sudo cloud-init clean --logs

# 4. SSH host keys are removed (will regenerate on first boot)
sudo rm -f /etc/ssh/ssh_host_*

# 5. Hostname is generic or cloud-init-managed
hostname
# Should be "ubuntu" or "template-vm", not a production hostname
```

## Conclusion

VM templates for IPv6 environments should use cloud-init with `dhcp6: true` or `accept-ra: true` rather than static IPv6 addresses, ensuring each cloned VM receives a unique IPv6 address via SLAAC or DHCPv6. The machine-id must be cleared in templates so each VM gets a unique DUID for DHCPv6. Packer automates the template build process and can install cloud-init configurations as part of the build. VMware provides a customization spec mechanism for injecting IPv6 addresses at clone time. The cloud-init network configuration `version: 2` with `dhcp6: true` is the most portable approach across KVM, VMware, Hyper-V, and cloud providers.
