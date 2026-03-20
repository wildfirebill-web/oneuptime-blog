# How to Inspect Helm Chart Details in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Helm, Kubernetes, Chart Management, DevOps

Description: Learn how to inspect deployed Helm chart releases, view values, check manifests, and review history in Portainer.

## Overview

After deploying a Helm chart, you often need to inspect the deployed release — to see what values were used, what manifests were generated, or to check the release history. Portainer exposes these details through the UI.

## Accessing Helm Release Details

1. Select your Kubernetes environment.
2. Navigate to **Applications > Helm charts**.
3. Click on a deployed release to open its detail view.

## What You Can See

### Release Overview
- **Chart name and version**
- **App version** (the application version, separate from chart version)
- **Namespace**
- **Status** (deployed, failed, pending)
- **Last deployed** timestamp

### User-Supplied Values

The values section shows the values used for this deployment — the combination of chart defaults and your overrides:

```yaml
# Example of values visible in the release detail
replicaCount: 3
service:
  type: LoadBalancer
  port: 80
ingress:
  enabled: true
  hostname: myapp.example.com
```

### Generated Manifests

Portainer shows the actual Kubernetes manifests that Helm generated and applied to the cluster. This is invaluable for debugging.

### Release History

```bash
# Equivalent CLI command to view release history
helm history my-release --namespace production

# Check current values applied
helm get values my-release --namespace production

# Check all manifests applied by the release
helm get manifest my-release --namespace production

# Get full release info
helm get all my-release --namespace production
```

## Comparing Values Across Revisions

```bash
# Get values from a specific revision
helm get values my-release --revision 2 --namespace production

# Compare with current values
helm get values my-release --namespace production
```

## Checking Chart Notes

Helm charts include post-install notes with important information (access URLs, next steps):

```bash
# View the chart notes
helm get notes my-release --namespace production
```

## Rolling Back to a Previous Version

If an upgrade caused issues, you can roll back from either the CLI or Portainer's UI:

```bash
# List available revisions
helm history my-release --namespace production

# Roll back to a specific revision
helm rollback my-release 2 --namespace production
```

In Portainer, the release detail page shows a history table with a **Rollback** button for each revision.

## Conclusion

Inspecting Helm chart details in Portainer gives you visibility into what was deployed, with what configuration, and when. This audit trail is essential for troubleshooting deployment issues and understanding the current state of your cluster.
