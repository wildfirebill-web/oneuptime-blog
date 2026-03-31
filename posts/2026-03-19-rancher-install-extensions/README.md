# How to Install Rancher UI Extensions

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, UI Extensions, Plugin, Dashboard

Description: Learn how to install, manage, and troubleshoot Rancher UI extensions to add new features and integrations to your Rancher dashboard.

Rancher UI Extensions allow you to add new pages, resource views, and integrations to the Rancher dashboard. This guide covers how to enable the extensions feature, install extensions from official and third-party repositories, and manage their lifecycle.

## Prerequisites

- Rancher v2.7.0 or later
- Admin or cluster-owner privileges
- A Rancher instance with internet access (or a private registry for air-gapped environments)

## Enabling the Extensions Feature

Extensions may need to be enabled on your Rancher instance before you can install them.

### Step 1: Enable Extensions in the UI

1. Log into Rancher as an administrator
2. Navigate to the main menu and look for **Extensions** in the sidebar
3. If you see a prompt to enable extensions, click **Enable**
4. Rancher will install the `ui-plugin-operator` in the `cattle-ui-plugin-system` namespace

### Step 2: Verify the Operator is Running

```bash
kubectl get pods -n cattle-ui-plugin-system
```

You should see the `ui-plugin-operator` pod running:

```plaintext
NAME                                   READY   STATUS    RESTARTS   AGE
ui-plugin-operator-xxxxxxxxx-xxxxx     1/1     Running   0          2m
```

### Step 3: Enable via Helm (Alternative)

If you prefer to enable extensions via Helm:

```bash
helm upgrade rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set ui-plugin-operator.enabled=true \
  --reuse-values
```

## Installing Extensions from the Built-In Repository

### Step 1: Browse Available Extensions

1. Click **Extensions** in the Rancher sidebar
2. Switch to the **Available** tab
3. Browse the list of available extensions

### Step 2: Install an Extension

1. Find the extension you want to install
2. Click the **Install** button
3. Select the version you want to install
4. Click **Install** to confirm

The extension is downloaded and loaded into the Rancher UI. A page refresh may be required.

### Step 3: Verify Installation

After installation, the extension appears in the **Installed** tab. Navigate to the new menu items or features it adds.

## Adding Third-Party Extension Repositories

### Step 1: Add a Helm Repository via the UI

1. Go to **Extensions** in the sidebar
2. Click the kebab menu (three dots) in the top right
3. Select **Manage Repositories**
4. Click **Create**
5. Fill in the repository details:
   - **Name**: A unique identifier (e.g., `my-company-extensions`)
   - **Index URL**: The Helm repository URL (e.g., `https://charts.example.com`)
6. Click **Create**

### Step 2: Add a Repository via kubectl

```yaml
# extension-repo.yaml

apiVersion: catalog.cattle.io/v1
kind: ClusterRepo
metadata:
  name: my-company-extensions
spec:
  url: https://charts.example.com
```

```bash
kubectl apply -f extension-repo.yaml
```

### Step 3: Add an OCI-Based Repository

For OCI container registries:

```yaml
apiVersion: catalog.cattle.io/v1
kind: ClusterRepo
metadata:
  name: oci-extensions
spec:
  url: oci://registry.example.com/charts
```

### Step 4: Install from the New Repository

After adding the repository, return to the **Extensions** page. Your new extensions should appear in the **Available** tab after the repository syncs (this may take a few minutes).

## Installing Extensions via the API

### List Available Extensions

```bash
export RANCHER_URL="https://rancher.example.com"
export RANCHER_TOKEN="token-xxxxx:yyyyyyyyyyyyyyyy"

curl -s -k \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  "${RANCHER_URL}/v1/catalog.cattle.io.clusterrepos" | jq '.data[] | {name: .metadata.name, url: .spec.url}'
```

### Install an Extension via Helm

```bash
# Add the extension chart repository
helm repo add rancher-extensions https://charts.rancher.io/extensions
helm repo update

# Install the extension
helm install my-extension rancher-extensions/my-extension \
  --namespace cattle-ui-plugin-system \
  --version 1.0.0
```

### Install Using the UIPlugin Custom Resource

```yaml
# ui-plugin.yaml
apiVersion: catalog.cattle.io/v1
kind: UIPlugin
metadata:
  name: my-extension
  namespace: cattle-ui-plugin-system
spec:
  plugin:
    name: my-extension
    version: 1.0.0
    endpoint: https://charts.example.com/extensions/my-extension/1.0.0
    noCache: false
    noAuth: false
```

```bash
kubectl apply -f ui-plugin.yaml
```

## Managing Installed Extensions

### Viewing Installed Extensions

Via the UI:

1. Go to **Extensions** in the sidebar
2. Check the **Installed** tab

Via kubectl:

```bash
kubectl get uiplugins -n cattle-ui-plugin-system
```

Via the API:

```bash
curl -s -k \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  "${RANCHER_URL}/v1/catalog.cattle.io.uiplugins" | jq '.data[] | {
    name: .metadata.name,
    version: .spec.plugin.version,
    state: .status.cacheState
  }'
```

### Upgrading Extensions

#### Via the UI

1. Go to **Extensions > Installed**
2. If an update is available, an **Update** button appears
3. Click **Update** and select the new version
4. Confirm the upgrade

#### Via Helm

```bash
helm upgrade my-extension rancher-extensions/my-extension \
  --namespace cattle-ui-plugin-system \
  --version 2.0.0
```

### Disabling Extensions

You can disable an extension without uninstalling it:

```bash
kubectl patch uiplugin my-extension -n cattle-ui-plugin-system \
  --type merge -p '{"spec": {"plugin": {"noCache": true}}}'
```

### Uninstalling Extensions

#### Via the UI

1. Go to **Extensions > Installed**
2. Click the extension's kebab menu
3. Select **Uninstall**
4. Confirm the removal

#### Via Helm

```bash
helm uninstall my-extension -n cattle-ui-plugin-system
```

#### Via kubectl

```bash
kubectl delete uiplugin my-extension -n cattle-ui-plugin-system
```

## Air-Gapped Installation

For environments without internet access, you need to pre-load extension assets.

### Step 1: Download Extension Assets

On a machine with internet access:

```bash
# Pull the extension chart
helm pull rancher-extensions/my-extension --version 1.0.0

# Download any container images referenced by the extension
docker pull registry.example.com/extensions/my-extension:1.0.0
```

### Step 2: Push to Internal Registry

```bash
# Push chart to internal Helm repository
helm push my-extension-1.0.0.tgz oci://internal-registry.example.com/charts

# Push images to internal registry
docker tag registry.example.com/extensions/my-extension:1.0.0 \
  internal-registry.example.com/extensions/my-extension:1.0.0
docker push internal-registry.example.com/extensions/my-extension:1.0.0
```

### Step 3: Configure Rancher to Use Internal Repository

```yaml
apiVersion: catalog.cattle.io/v1
kind: ClusterRepo
metadata:
  name: internal-extensions
spec:
  url: oci://internal-registry.example.com/charts
```

## Troubleshooting

### Extension Not Appearing After Installation

1. Check the UIPlugin resource status:

```bash
kubectl describe uiplugin my-extension -n cattle-ui-plugin-system
```

2. Check the operator logs:

```bash
kubectl logs -n cattle-ui-plugin-system -l app=ui-plugin-operator --tail=50
```

3. Clear your browser cache and reload the Rancher UI.

### Extension Loading Errors

Check the browser console (F12) for JavaScript errors. Common causes:
- Version incompatibility between the extension and Rancher
- Missing dependencies
- CORS issues with external resources

### Repository Sync Failures

```bash
# Check repository status
kubectl get clusterrepos -o jsonpath='{.items[*].status.conditions}'

# Force a repository refresh
kubectl annotate clusterrepo my-repo cattle.io/force-update=$(date +%s) --overwrite
```

### Permissions Issues

Ensure the user has the correct permissions:

```bash
# Check if the user can manage UI plugins
kubectl auth can-i create uiplugins --namespace cattle-ui-plugin-system --as=user-xxxxx
```

## Summary

Installing Rancher UI extensions involves enabling the extension operator, adding extension repositories, and installing extensions through the UI, Helm, or kubectl. For air-gapped environments, pre-download extension assets and configure internal repositories. Keep extensions updated for security and compatibility, and use the UIPlugin custom resource for declarative management.
