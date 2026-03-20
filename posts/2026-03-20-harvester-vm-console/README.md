# How to Access VM Console in Harvester

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Harvester, VM Console, Virtual Machine, KubeVirt, Kubernetes, SUSE Rancher, HCI

Description: Learn how to access the console of a virtual machine running in Harvester using the web-based VNC console, kubectl virt plugin, and serial console for headless VMs.

---

Harvester provides multiple ways to access virtual machine consoles, whether for GUI-based VMs (using VNC) or headless server VMs (using serial console). Console access is essential for troubleshooting VMs that have lost network connectivity.

---

## Method 1: Web Console (VNC) via Harvester UI

The simplest way to access a VM console is through the Harvester web interface:

1. Navigate to **Virtual Machines** in the Harvester dashboard
2. Click on the VM you want to access
3. Click the **Console** button (or the terminal icon)
4. A VNC console opens in your browser
5. Click inside the console window to capture keyboard input

This method works for both Linux and Windows VMs.

---

## Method 2: kubectl virt Console

For command-line access, use the `virtctl` tool:

```bash
# Install virtctl

VERSION=$(kubectl get kubevirt -n harvester-system -o jsonpath='{.items[0].status.observedKubeVirtVersion}')
curl -Lo virtctl https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/virtctl-${VERSION}-linux-amd64
chmod +x virtctl
sudo mv virtctl /usr/local/bin/

# Connect to VM console (VNC via terminal)
virtctl vnc <vm-name> -n <namespace>

# Connect to serial console (text-only, no VNC needed)
virtctl console <vm-name> -n <namespace>
```

---

## Method 3: Enable Serial Console on Linux VMs

For headless VMs, enable the serial console so it's accessible via `virtctl console`:

```bash
# Inside the VM, add serial console to the kernel
# For GRUB-based systems (CentOS, RHEL, Ubuntu):
grubby --update-kernel=ALL --args="console=ttyS0,115200n8"
# or
sed -i 's/GRUB_CMDLINE_LINUX="/GRUB_CMDLINE_LINUX="console=ttyS0,115200n8 /' /etc/default/grub
grub2-mkconfig -o /boot/grub2/grub.cfg

# Enable and start the getty service
systemctl enable serial-getty@ttyS0.service
systemctl start serial-getty@ttyS0.service

# Reboot the VM
reboot
```

After the VM reboots, connect via:

```bash
virtctl console <vm-name> -n <namespace>
```

---

## Method 4: Access VM Console via SSH Tunnel

If the Harvester API server is not directly accessible:

```bash
# Create an SSH tunnel to the Harvester node
ssh -L 8443:localhost:443 user@harvester-node

# Then use virtctl with the local API server
export KUBECONFIG=/path/to/harvester-kubeconfig
virtctl vnc <vm-name> -n <namespace>
```

---

## Step 5: Configure VM for Console from the VirtualMachine Spec

```yaml
# Add serial console device to an existing VM
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: my-vm
  namespace: default
spec:
  template:
    spec:
      domain:
        devices:
          # Enable serial console
          serial:
            - tag: serial0
              name: serial0
          # Enable VNC console
          autoattachGraphicsDevice: true
```

---

## Troubleshooting Console Access

```bash
# Check if the VM is running
kubectl get vmi -n <namespace> <vm-name>
# VMI status should be: Running

# Check if virt-vnc service is running
kubectl get pods -n harvester-system | grep virt-vnc

# Check KubeVirt virt-handler on the node running the VM
NODE=$(kubectl get vmi -n <namespace> <vm-name> -o jsonpath='{.status.nodeName}')
kubectl get pod -n harvester-system \
  -l kubevirt.io=virt-handler \
  --field-selector spec.nodeName=$NODE

# View virt-handler logs for console errors
kubectl logs -n harvester-system \
  $(kubectl get pod -n harvester-system \
    -l kubevirt.io=virt-handler \
    --field-selector spec.nodeName=$NODE -o name)
```

---

## Best Practices

- Always enable the serial console on Linux server VMs at provisioning time - if the VM loses network access, the serial console is the only way to recover it without a reboot.
- Use `virtctl console` for automation scripts that need to interact with a VM - it's scriptable unlike the web VNC console.
- For Windows VMs, the VNC web console is the only practical option - ensure the VM has a VGA-compatible display device configured.
