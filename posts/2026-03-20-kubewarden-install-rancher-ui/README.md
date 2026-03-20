# How to Install Kubewarden from Rancher UI

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubewarden, Rancher, Kubernetes, Policy, Security

Description: Learn how to install and configure Kubewarden directly from the Rancher UI using the Rancher Apps catalog for simplified policy enforcement setup.

## Introduction

Rancher provides a graphical way to install Kubewarden through its integrated app catalog. This approach simplifies the installation process significantly — you can deploy Kubewarden without writing any Helm commands, configure all options through forms, and manage updates through the Rancher interface.

This guide covers the complete Kubewarden installation process using the Rancher UI, from prerequisites through policy server verification.

## Prerequisites

- Rancher v2.7 or later
- A Kubernetes cluster managed by Rancher
- Cluster Admin role in Rancher
- Internet access from the cluster nodes (or local charts configured)

## Step 1: Enable the Rancher Charts Repository

Kubewarden charts are available through the Rancher charts repository or Kubewarden's own Helm chart repository.

### Adding Kubewarden to Rancher Charts

1. In Rancher, navigate to the cluster where you want to install Kubewarden
2. Click **Apps > Repositories** in the left sidebar
3. Click **Create**
4. Configure:
   - **Name**: `kubewarden`
   - **Target**: `https://charts.kubewarden.io`
   - **Branch**: `main` (or leave as default)
5. Click **Create**

Wait for the repository to sync (indicated by a green checkmark).

## Step 2: Install Cert-Manager (If Not Already Installed)

Kubewarden requires cert-manager. Check if it's already installed:

1. Navigate to **Apps > Installed Apps**
2. Search for "cert-manager"
3. If not found, install it:
   - Navigate to **Apps > Charts**
   - Search for "cert-manager"
   - Click **Install**
   - Select namespace: `cert-manager`
   - Check **Create Namespace**
   - Enable **Install CRDs**
   - Click **Next** then **Install**

## Step 3: Install Kubewarden CRDs

1. Navigate to **Apps > Charts**
2. Search for "kubewarden-crds"
3. Click on the `kubewarden-crds` chart
4. Click **Install**
5. Configure:
   - **Namespace**: `kubewarden`
   - Check **Create Namespace**
6. Click **Next**
7. Review the configuration (no required changes for defaults)
8. Click **Install**

Wait for the installation to complete (all resources show as active).

## Step 4: Install the Kubewarden Controller

1. Navigate to **Apps > Charts**
2. Search for "kubewarden-controller"
3. Click on the chart
4. Click **Install**
5. Configure:
   - **Namespace**: `kubewarden` (same namespace as CRDs)
   - **Chart Version**: Select the latest stable version
6. Click **Next**
7. In the values editor, you can customize:
   ```yaml
   # Optional: Enable HA for the controller
   controller:
     replicaCount: 2
   ```
8. Click **Install**

## Step 5: Install Kubewarden Defaults (Policy Server)

1. Navigate to **Apps > Charts**
2. Search for "kubewarden-defaults"
3. Click on the chart
4. Click **Install**
5. Configure:
   - **Namespace**: `kubewarden`
6. Click **Next**
7. In the values editor, optionally configure:
   ```yaml
   # Policy server replica count for HA
   policyServer:
     replicaCount: 2

   # Resource limits for the policy server
   policyServer:
     resources:
       requests:
         cpu: "200m"
         memory: "256Mi"
       limits:
         cpu: "1"
         memory: "1Gi"
   ```
8. Click **Install**

## Step 6: Verify the Installation

### Via Rancher UI

1. Navigate to **Apps > Installed Apps**
2. Verify all three Kubewarden apps show as **Deployed**:
   - `kubewarden-crds`
   - `kubewarden-controller`
   - `kubewarden-defaults`

### Via kubectl

```bash
# Check all Kubewarden pods
kubectl get pods -n kubewarden

# Verify the PolicyServer resource
kubectl get policyserver

# Check the ValidatingWebhookConfiguration
kubectl get validatingwebhookconfigurations | grep kubewarden
```

## Step 7: Apply Your First Policy via Rancher UI

After installation, you can apply policies directly from the Rancher UI:

1. Navigate to the cluster in Rancher
2. Look for **Policy** or **Kubewarden** in the sidebar
   (available if the Rancher Kubewarden extension is installed)
3. Or use **More Resources > policies.kubewarden.io > ClusterAdmissionPolicies**
4. Click **Create**

Alternatively, create a policy via kubectl:

```bash
# Apply a sample policy to test the installation
kubectl apply -f - <<EOF
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: no-privileged-pods
spec:
  module: registry://ghcr.io/kubewarden/policies/pod-privileged:v0.2.0
  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations:
        - CREATE
        - UPDATE
  mutating: false
EOF

# Check policy status
kubectl get clusteradmissionpolicy no-privileged-pods
```

## Upgrading Kubewarden from Rancher UI

1. Navigate to **Apps > Installed Apps**
2. Find a Kubewarden app showing an upgrade is available
3. Click the three-dot menu
4. Click **Edit/Upgrade**
5. Review the new version and configuration
6. Click **Upgrade**

Upgrade in this order:
1. `kubewarden-crds`
2. `kubewarden-controller`
3. `kubewarden-defaults`

## Accessing Kubewarden Logs from Rancher

1. Navigate to your cluster in Rancher
2. Go to **Workloads > Pods**
3. Select namespace: `kubewarden`
4. Click on the policy server pod
5. Click **Logs** to view real-time logs

## Conclusion

Installing Kubewarden through the Rancher UI provides a streamlined experience that eliminates the need to manage Helm commands directly. The multi-step installation process (CRDs → Controller → Defaults) ensures all components are installed in the correct order. Once installed, Kubewarden integrates naturally with Rancher's monitoring and logging capabilities, giving you a complete policy enforcement platform accessible through the familiar Rancher interface.
