# How to Troubleshoot Harvester Installation Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Harvester, Troubleshooting, Installation, HCI, Kubernetes, SUSE Rancher

Description: Learn how to diagnose and resolve common Harvester installation failures including network configuration issues, storage initialization errors, and node join problems.

---

Harvester installation can fail due to hardware incompatibilities, network misconfigurations, or BIOS settings. This guide covers the most common failure scenarios and their solutions.

---

## Step 1: Check Installation Logs

During ISO installation, access the console logs:

```bash
# At the Harvester console, press CTRL+ALT+F2 to switch to a shell

# View installation logs
journalctl -f

# Check cloud-init log
cat /var/log/cloud-init-output.log

# Check Harvester startup
journalctl -u rke2-server -f
```

---

## Issue 1: Installation Hangs at "Starting Services"

**Cause**: Usually a network interface naming issue or VLAN misconfiguration.

```bash
# Check available network interfaces
ip link show

# Check if the management interface was configured correctly
cat /etc/rancher/rke2/config.yaml | grep node-ip

# If the IP is wrong, reconfigure via the Harvester TUI
harvester-config
```

---

## Issue 2: Nodes Cannot Join the Cluster

**Symptoms**: Additional nodes show "Join Token Invalid" or timeout.

```bash
# On the first node, get the correct join token
cat /var/lib/rancher/rke2/server/node-token

# Verify the token matches what was entered during node setup
# Check network connectivity from joining node to first node
curl -k https://<first-node-ip>:9345/ping
```

---

## Issue 3: Storage Not Initializing

**Symptoms**: Longhorn fails to start, disks not detected.

```bash
# Check if disks are visible
lsblk

# Check Longhorn manager pod status
kubectl get pods -n longhorn-system

# Common issue: disk is already partitioned
# Wipe it if you're sure it's safe
wipefs -a /dev/sdb
```

---

## Issue 4: Harvester WebUI Not Accessible

```bash
# Check if the nginx ingress is running
kubectl get pods -n harvester-system

# Check the VIP (virtual IP) is configured correctly
kubectl get svc -n kube-system | grep kube-vip

# Verify the management IP is assigned
ip addr show

# Check the Harvester ingress
kubectl get ingress -n harvester-system
```

---

## Issue 5: After Installation, Cluster Shows "Not Ready"

```bash
# Check RKE2 server status
systemctl status rke2-server

# Check all system pods
kubectl get pods -A | grep -v Running

# Check events for errors
kubectl get events -A --sort-by=.lastTimestamp | tail -30

# Verify etcd health
/var/lib/rancher/rke2/bin/etcdctl \
  --endpoints https://127.0.0.1:2379 \
  --cacert /var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
  --cert /var/lib/rancher/rke2/server/tls/etcd/client.crt \
  --key /var/lib/rancher/rke2/server/tls/etcd/client.key \
  endpoint health
```

---

## Issue 6: BIOS/UEFI Configuration Problems

Common hardware requirements not met:

```text
- VT-x / AMD-V virtualization must be enabled in BIOS
- IOMMU must be enabled (for PCI passthrough)
- Secure Boot must be disabled (Harvester doesn't support Secure Boot)
- RAID controller must be in AHCI mode or pass-through
```

---

## Generating a Support Bundle

If the issue persists:

```bash
# Via Harvester UI: Support > Download Support Bundle
# Or via kubectl:
kubectl get supportbundle -A
```

---

## Best Practices

- Verify all hardware meets the Harvester minimum requirements before installation.
- Use the official Harvester ISO from the releases page - custom or modified ISOs may behave unpredictably.
- Check the Harvester compatibility list for your NIC and storage controller models.
