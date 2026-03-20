# How to Upgrade RKE2 Clusters

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RKE2, Upgrade, Kubernetes, Cluster Management, SUSE Rancher, Maintenance

Description: Learn how to upgrade RKE2 clusters to a new Kubernetes version, covering pre-upgrade checks, the upgrade process for server and agent nodes, and post-upgrade validation.

---

Upgrading RKE2 is a two-phase process: first upgrade server (control plane) nodes, then upgrade agent (worker) nodes. RKE2 supports in-place upgrades without reprovisioning nodes.

---

## Pre-Upgrade Checklist

```bash
# 1. Verify current version
rke2 --version

# 2. Check all nodes are Ready
kubectl get nodes

# 3. Check no pods are in CrashLoopBackOff
kubectl get pods -A | grep -v Running | grep -v Completed

# 4. Back up etcd
rke2 etcd-snapshot save --name pre-upgrade-$(date +%Y%m%d)

# 5. Review the target release notes
# https://github.com/rancher/rke2/releases
```

---

## Method 1: Upgrade via Rancher UI (Recommended)

If the cluster is managed by Rancher:

1. Go to **Cluster Management > Your Cluster > Edit Config**
2. Change **Kubernetes Version** to the target version
3. Configure upgrade strategy (concurrency, drain settings)
4. Click **Save**

Rancher will handle the rolling upgrade automatically.

---

## Method 2: Manual Upgrade via RKE2 Install Script

### Upgrade Server Nodes

Upgrade one server node at a time. Start with a non-leader node:

```bash
# On each server node (one at a time)
# Download and install the new version
curl -sfL https://get.rke2.io | \
  INSTALL_RKE2_VERSION="v1.30.2+rke2r1" \
  sh -

# Restart the service
systemctl restart rke2-server

# Verify the node upgraded
kubectl get node $(hostname) -o jsonpath='{.status.nodeInfo.kubeletVersion}'
```

Wait for the node to return to `Ready` before upgrading the next server node.

---

### Upgrade Agent Nodes

After all server nodes are upgraded, upgrade agents:

```bash
# On each agent node
# Optionally drain the node first
kubectl drain <node-name> \
  --ignore-daemonsets \
  --delete-emptydir-data

# Install new version
curl -sfL https://get.rke2.io | \
  INSTALL_RKE2_TYPE="agent" \
  INSTALL_RKE2_VERSION="v1.30.2+rke2r1" \
  sh -

# Restart
systemctl restart rke2-agent

# Uncordon after upgrade
kubectl uncordon <node-name>
```

---

## Method 3: System Upgrade Controller (GitOps)

For automated upgrades, use the System Upgrade Controller:

```bash
kubectl apply -f \
  https://github.com/rancher/system-upgrade-controller/releases/latest/download/system-upgrade-controller.yaml
```

Create upgrade plans (see the rolling upgrade guide for full details): https://oneuptime.com/blog/post/2026-03-20-rolling-upgrade-rke2/view

---

## Post-Upgrade Validation

```bash
# Confirm all nodes are on the new version
kubectl get nodes -o wide

# Check that all system pods are healthy
kubectl get pods -n kube-system

# Run a quick workload test
kubectl run test-pod --image=nginx --rm -it -- curl -s http://localhost

# Re-run CIS benchmark if applicable
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
```

---

## Best Practices

- Only upgrade one minor version at a time (e.g., 1.28 → 1.29, not 1.28 → 1.30).
- Test upgrades in a staging cluster that mirrors production before upgrading production.
- Keep the etcd backup for at least 7 days after a successful upgrade before deleting it.
- Upgrade RKE2 before upgrading Rancher — check Rancher's support matrix for compatible versions.
