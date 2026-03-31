# How to Configure Longhorn Support Bundle Manager - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Longhorn, Support Bundle, Troubleshooting, Kubernetes, Diagnostic, SUSE Rancher

Description: Learn how to use Longhorn's support bundle feature to collect comprehensive diagnostic information including logs, configurations, and volume status for troubleshooting and support escalations.

---

Longhorn's support bundle feature collects all diagnostic information - pod logs, settings, volume status, node conditions, and Kubernetes events - into a single archive. This makes it easy to share relevant information with the Longhorn team or your support provider.

---

## What the Support Bundle Collects

- Longhorn manager, driver, and engine pod logs
- Kubernetes events in the `longhorn-system` namespace
- All Longhorn CRD objects (volumes, replicas, nodes, settings)
- Node and disk status
- Longhorn version information
- Cluster node information

---

## Step 1: Generate a Support Bundle via the UI

The simplest way is through the Longhorn UI:

1. Navigate to **Support**
2. Click **Generate Support Bundle**
3. Enter a description (e.g., "Volume XXX stuck in attaching state")
4. Click **Generate** - the bundle is created as a zip archive
5. Download the bundle

---

## Step 2: Generate a Support Bundle via kubectl

```yaml
# Create a SupportBundle resource

apiVersion: longhorn.io/v1beta2
kind: SupportBundle
metadata:
  name: support-bundle-2026-03-20
  namespace: longhorn-system
spec:
  description: "Volume pvc-xxxxx stuck in attaching state"
  # Error logs only saves space; set to false for full logs
  issueURL: ""
```

```bash
kubectl apply -f support-bundle.yaml

# Watch the bundle generation progress
kubectl get lhsupportbundle -n longhorn-system -w

# When status shows "ReadyForDownload", get the download URL
kubectl get lhsupportbundle support-bundle-2026-03-20 \
  -n longhorn-system \
  -o jsonpath='{.status.fileLocation}'
```

---

## Step 3: Download the Support Bundle

```bash
# Port-forward the Longhorn frontend
kubectl port-forward -n longhorn-system svc/longhorn-frontend 8080:80

# Download the bundle
BUNDLE_NAME="support-bundle-2026-03-20"
curl -o support-bundle.zip \
  "http://localhost:8080/v1/supportbundles/${BUNDLE_NAME}/download"
```

---

## Step 4: Inspect the Support Bundle

```bash
# Extract the bundle
unzip support-bundle.zip -d support-bundle/

# View the directory structure
ls support-bundle/

# Check manager logs for errors
grep -i "error\|fatal\|panic" support-bundle/logs/longhorn-manager-*/

# Check volume status at time of issue
cat support-bundle/longhorn-crds/volumes.json | jq '.items[] | {name:.metadata.name, state:.status.state}'
```

---

## Step 5: Clean Up Old Support Bundles

```bash
# Delete old support bundle resources
kubectl delete lhsupportbundle support-bundle-2026-03-20 -n longhorn-system

# The archive files are deleted automatically when the resource is deleted
```

---

## Best Practices

- Generate a support bundle **immediately** when you notice an issue - logs are rotated and historical data may be lost if you wait.
- Include a clear description of the issue in the bundle so support engineers understand the context.
- Share the bundle via a private channel - it contains cluster topology and configuration information.
- Run support bundle generation before performing risky operations (upgrades, node removals) as a baseline snapshot.
