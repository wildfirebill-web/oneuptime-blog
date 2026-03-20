# How to Configure Harvester SR-IOV for Network Performance - Networking

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Harvester, SR-IOV, Networking, High Performance, HCI, KubeVirt, SUSE Rancher

Description: Learn how to configure SR-IOV (Single Root I/O Virtualization) in Harvester to provide VMs with near-native network performance by bypassing the software switching layer.

---

SR-IOV allows a single physical NIC to present multiple virtual functions (VFs) directly to VMs, bypassing the Kubernetes software bridge and achieving near-native network throughput with sub-microsecond latency.

---

## Prerequisites

- SR-IOV capable NIC (Intel X710, Mellanox ConnectX series, etc.)
- BIOS with SR-IOV and IOMMU enabled
- Linux kernel 4.15+ with `igb`, `ixgbe`, or `mlx5_core` drivers

---

## Step 1: Enable IOMMU and SR-IOV in BIOS

In your server BIOS/UEFI:
- Enable **VT-d** (Intel) or **AMD-Vi** (AMD) for IOMMU
- Enable **SR-IOV** support
- Save and reboot

Verify IOMMU is active:

```bash
dmesg | grep -e DMAR -e IOMMU | head -5
# Should show: "IOMMU enabled"

```

---

## Step 2: Configure SR-IOV Virtual Functions

On each Harvester node with SR-IOV NICs:

```bash
# Find the physical NIC (PF - Physical Function)
lspci | grep -i ethernet

# Check the NIC supports SR-IOV
cat /sys/bus/pci/devices/<pci-address>/sriov_totalvfs

# Create virtual functions
# Replace ens3 with your NIC name and 8 with desired VF count
echo 8 > /sys/bus/pci/devices/<pci-address>/sriov_numvfs

# Make persistent via udev rule
cat > /etc/udev/rules.d/99-sriov.rules <<EOF
ACTION=="add", SUBSYSTEM=="net", KERNEL=="ens3", RUN+="/bin/sh -c 'echo 8 > /sys/bus/pci/devices/\$ID_PATH_TAG/sriov_numvfs'"
EOF

# Verify VFs were created
ip link show ens3
```

---

## Step 3: Install SR-IOV Network Device Plugin

```bash
# Install the SR-IOV device plugin as a DaemonSet
kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/sriov-network-device-plugin/master/deployments/sriovdp-daemonset.yaml

# Create a ConfigMap specifying which VFs to expose
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: sriovdp-config
  namespace: kube-system
data:
  config.json: |
    {
      "resourceList": [
        {
          "resourceName": "intel_sriov_netdevice",
          "selectors": {
            "vendors": ["8086"],
            "devices": ["154c"],
            "drivers": ["iavf"]
          }
        }
      ]
    }
EOF
```

---

## Step 4: Create a Network Attachment Definition for SR-IOV

```yaml
# sriov-net-attach-def.yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: sriov-dpdk
  namespace: default
  annotations:
    k8s.v1.cni.cncf.io/resourceName: intel.com/intel_sriov_netdevice
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "name": "sriov-dpdk",
      "type": "sriov",
      "spoofchk": "off",
      "trust": "on"
    }
```

---

## Step 5: Attach SR-IOV Network to a VM

```yaml
# vm-with-sriov.yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: high-perf-vm
  namespace: default
spec:
  running: true
  template:
    spec:
      networks:
        - name: default
          pod: {}
        - name: sriov-net
          multus:
            networkName: sriov-dpdk
      domain:
        resources:
          requests:
            # Request SR-IOV VF
            intel.com/intel_sriov_netdevice: "1"
        devices:
          interfaces:
            - name: default
              masquerade: {}
            - name: sriov-net
              sriov: {}   # SR-IOV interface
```

---

## Best Practices

- Use SR-IOV for latency-sensitive workloads like financial trading systems or HPC applications.
- Keep at least 2 VFs per node for cluster management traffic - do not allocate all VFs to VMs.
- Monitor VF utilization using `ethtool -S ens3` or the DPDK PMD stats.
