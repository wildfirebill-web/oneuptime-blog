# How to Upgrade Rancher to the Latest Version

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Upgrade

Description: A step-by-step guide to upgrading your Rancher installation to the latest stable version using Helm.

Keeping Rancher up to date is essential for security patches, bug fixes, and access to new features. This guide walks you through upgrading Rancher to the latest version on a Kubernetes cluster using Helm, covering prerequisites, backup, and the actual upgrade process.

## Prerequisites

Before you begin, make sure you have the following in place:

- A running Rancher installation deployed via Helm on a Kubernetes cluster
- `kubectl` configured to access the cluster running Rancher
- Helm 3 installed on your workstation
- Administrative access to the Rancher UI
- A recent backup of your Rancher data

## Step 1: Check Your Current Rancher Version

First, verify the version of Rancher you are currently running. You can do this through the Rancher UI by navigating to the bottom-left corner of the dashboard, or via the command line:

```bash
kubectl get settings server-version -o jsonpath='{.value}' -n cattle-system
```

You can also check the Helm release:

```bash
helm list -n cattle-system
```

This will show the chart version and app version currently deployed.

## Step 2: Review the Release Notes

Visit the official Rancher releases page and review the release notes for every version between your current version and the target version. Pay attention to:

- Breaking changes
- Deprecations
- New requirements for Kubernetes versions
- Any manual migration steps

## Step 3: Back Up Your Rancher Installation

Before any upgrade, create a backup. If you are using Rancher on an RKE cluster, take an etcd snapshot:

```bash
rke etcd snapshot-save --config cluster.yml --name pre-upgrade-snapshot
```

If you are running Rancher on RKE2 or K3s, use the built-in snapshot mechanism:

```bash
# For RKE2
rke2 etcd-snapshot save --name pre-upgrade-snapshot

# For K3s
k3s etcd-snapshot save --name pre-upgrade-snapshot
```

You should also back up any custom resources and configurations:

```bash
kubectl get settings -n cattle-system -o yaml > rancher-settings-backup.yaml
kubectl get clusters.management.cattle.io -o yaml > clusters-backup.yaml
```

## Step 4: Update the Helm Repository

Add or update the Rancher Helm repository to make sure you have the latest charts:

```bash
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update
```

Verify the latest available version:

```bash
helm search repo rancher-stable/rancher --versions | head -10
```

## Step 5: Check Current Helm Values

Before upgrading, export your current Helm values so you can preserve your configuration:

```bash
helm get values rancher -n cattle-system -o yaml > current-values.yaml
```

Review the file to confirm your current settings such as hostname, ingress configuration, TLS settings, and replica count.

## Step 6: Perform the Upgrade

Run the Helm upgrade command, passing in your existing values:

```bash
helm upgrade rancher rancher-stable/rancher \
  --namespace cattle-system \
  --values current-values.yaml \
  --version <TARGET_VERSION>
```

If you want the absolute latest stable version, omit the `--version` flag:

```bash
helm upgrade rancher rancher-stable/rancher \
  --namespace cattle-system \
  --values current-values.yaml
```

## Step 7: Monitor the Upgrade

Watch the rollout progress:

```bash
kubectl rollout status deployment rancher -n cattle-system
```

Check that all Rancher pods are running and healthy:

```bash
kubectl get pods -n cattle-system
```

You should see the Rancher pods restarting with the new version. Wait until all pods show a `Running` status with all containers ready.

## Step 8: Verify the Upgrade

Confirm the new version is active:

```bash
kubectl get settings server-version -o jsonpath='{.value}' -n cattle-system
```

Log in to the Rancher UI and verify:

- The dashboard loads correctly
- All managed clusters show as active
- Cluster agents are connected and healthy
- Any custom catalogs and apps are functioning

## Step 9: Update Downstream Cluster Agents

After upgrading Rancher, the downstream cluster agents may need to be updated. Rancher typically handles this automatically, but you can verify by checking the agent versions in each managed cluster:

```bash
kubectl get deployments cattle-cluster-agent -n cattle-system -o jsonpath='{.spec.template.spec.containers[0].image}'
```

If agents are not updating automatically, you can trigger an update from the Rancher UI by navigating to each cluster and selecting the option to update the agent.

## Troubleshooting

If the upgrade fails or Rancher becomes unresponsive, check the pod logs:

```bash
kubectl logs -l app=rancher -n cattle-system --tail=100
```

Common issues include:

- **Certificate errors**: Make sure your TLS certificates are still valid and properly configured.
- **Webhook failures**: The Rancher webhook may need time to restart. Check its status with `kubectl get pods -n cattle-system | grep webhook`.
- **Resource limits**: Ensure the cluster has enough CPU and memory for the new version.

If the upgrade is completely broken, you can roll back using Helm:

```bash
helm rollback rancher -n cattle-system
```

Then restore your etcd snapshot if necessary.

## Best Practices

- Always upgrade one minor version at a time when possible. Do not skip multiple minor versions.
- Run upgrades during a maintenance window with low traffic.
- Test the upgrade in a staging environment before applying it to production.
- Keep your Kubernetes version compatible with the target Rancher version by checking the support matrix.
- Maintain a documented upgrade runbook for your team.

## Conclusion

Upgrading Rancher is straightforward when you follow the proper steps: back up your data, update the Helm repo, preserve your values, run the upgrade, and verify everything is working. By planning ahead and testing in staging, you can keep your Rancher installation current with minimal risk to your production workloads.
