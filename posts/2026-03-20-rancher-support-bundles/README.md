# How to Collect Rancher Support Bundles

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Troubleshooting, Support

Description: Learn how to collect comprehensive Rancher support bundles for cluster diagnostics, including logs, configurations, and cluster state using the Rancher UI and CLI tools.

## Introduction

When troubleshooting a complex Rancher issue or opening a support ticket with SUSE, a Support Bundle provides a comprehensive snapshot of your cluster's state: logs, resource manifests, events, and configuration. This guide covers how to collect support bundles through both the Rancher UI and command-line tools.

## What's in a Rancher Support Bundle?

A support bundle contains:
- Logs from Rancher server pods and agents
- Kubernetes resource manifests (Deployments, DaemonSets, ConfigMaps)
- Cluster events
- Node information (CPU, memory, OS version)
- RKE/RKE2 cluster configuration (with secrets redacted)
- Helm release history

## Method 1: Collect via the Rancher UI

1. Log in to Rancher as **admin**.
2. Navigate to **☰ → About** or **Global Settings → Support**.
3. Click **Download Support Bundle**.
4. Select the clusters to include.
5. Click **Generate** and wait for the bundle to be prepared.
6. Click **Download** to save the ZIP file.

## Method 2: Collect Using `rancher-support-bundle-kit`

SUSE provides an open-source tool for collecting support bundles:

```bash
# Install the support bundle kit
curl -Lo support-bundle-kit \
  https://github.com/rancher/support-bundle-kit/releases/latest/download/support-bundle-kit-linux-amd64
chmod +x support-bundle-kit
sudo mv support-bundle-kit /usr/local/bin/

# Collect a bundle (uses your current kubeconfig)
support-bundle-kit collect \
  --kubeconfig ~/.kube/config \
  --output-file /tmp/rancher-support-bundle.zip

# Collect with specific namespaces
support-bundle-kit collect \
  --namespaces cattle-system,cattle-provisioning-capi-system \
  --output-file /tmp/rancher-bundle-targeted.zip
```

## Method 3: Collect Manually with kubectl

For environments where the above tools aren't available:

```bash
#!/usr/bin/env bash
# manual-support-bundle.sh — Collect Rancher diagnostics manually

BUNDLE_DIR="/tmp/rancher-bundle-$(date +%Y%m%d-%H%M%S)"
mkdir -p "${BUNDLE_DIR}"/{logs,manifests,events}

echo "Collecting Rancher support bundle to ${BUNDLE_DIR}"

# 1. Collect pod logs from key namespaces
for ns in cattle-system cattle-provisioning-capi-system cattle-fleet-system; do
  echo "Collecting logs from namespace: ${ns}"
  mkdir -p "${BUNDLE_DIR}/logs/${ns}"
  for pod in $(kubectl get pods -n "${ns}" -o name); do
    pod_name=$(basename "${pod}")
    kubectl logs -n "${ns}" "${pod_name}" --all-containers \
      > "${BUNDLE_DIR}/logs/${ns}/${pod_name}.log" 2>&1 || true
    kubectl logs -n "${ns}" "${pod_name}" --all-containers --previous \
      > "${BUNDLE_DIR}/logs/${ns}/${pod_name}.previous.log" 2>&1 || true
  done
done

# 2. Collect resource manifests
kubectl get all -n cattle-system -o yaml \
  > "${BUNDLE_DIR}/manifests/cattle-system.yaml"
kubectl get nodes -o yaml \
  > "${BUNDLE_DIR}/manifests/nodes.yaml"
kubectl get events -A --sort-by='.lastTimestamp' \
  > "${BUNDLE_DIR}/events/all-events.txt"

# 3. Collect Rancher-specific resources
kubectl get clusters.management.cattle.io -o yaml \
  > "${BUNDLE_DIR}/manifests/clusters.yaml"
kubectl get settings.management.cattle.io -o yaml \
  > "${BUNDLE_DIR}/manifests/settings.yaml"

# 4. Collect node diagnostics
kubectl top nodes > "${BUNDLE_DIR}/node-resources.txt" 2>&1 || true
kubectl describe nodes > "${BUNDLE_DIR}/nodes-describe.txt"

# 5. Bundle everything
tar -czf "${BUNDLE_DIR}.tar.gz" -C "$(dirname ${BUNDLE_DIR})" "$(basename ${BUNDLE_DIR})"
echo "Bundle created: ${BUNDLE_DIR}.tar.gz"
```

```bash
chmod +x manual-support-bundle.sh
./manual-support-bundle.sh
```

## Method 4: Collect Cluster Agent Bundles

For downstream cluster issues, collect agent-specific diagnostics:

```bash
# Save to a directory
mkdir -p /tmp/agent-bundle

# Collect cattle-system resources from the DOWNSTREAM cluster
kubectl --kubeconfig /path/to/downstream/kubeconfig \
  get all,configmap,secret,events -n cattle-system -o yaml \
  > /tmp/agent-bundle/cattle-system.yaml

# Get agent logs
kubectl --kubeconfig /path/to/downstream/kubeconfig \
  logs -n cattle-system -l app=cattle-cluster-agent \
  > /tmp/agent-bundle/cattle-cluster-agent.log

kubectl --kubeconfig /path/to/downstream/kubeconfig \
  logs -n cattle-system -l app=cattle-agent \
  > /tmp/agent-bundle/cattle-node-agents.log
```

## What to Include When Opening a Support Ticket

When submitting to SUSE support or a GitHub issue, include:

1. The support bundle ZIP or tar.gz file.
2. Steps to reproduce the issue.
3. The Rancher version (`kubectl get setting server-version -o jsonpath='{.value}'`).
4. The Kubernetes distribution and version (RKE2, K3s, EKS, etc.).
5. Any recent changes (upgrades, cert rotations, node additions).

## Conclusion

Collecting a comprehensive support bundle is the first step toward resolving complex Rancher issues. The Rancher UI provides the easiest path, while `support-bundle-kit` and manual `kubectl` scripts offer flexibility in restricted environments. A well-collected support bundle dramatically reduces the time needed to diagnose and resolve issues, whether you're working with SUSE support or debugging independently.
