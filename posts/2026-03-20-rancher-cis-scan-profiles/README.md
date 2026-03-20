# How to Configure CIS Scan Profiles in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, CIS, Security, Compliance, Scan Profiles

Description: Learn how to create and customize CIS scan profiles in Rancher to tailor security benchmark scans to your organization's specific requirements.

Rancher's CIS scanning feature supports custom scan profiles that allow you to enable or disable specific benchmark checks based on your environment's requirements. This is essential when certain checks don't apply to your infrastructure or when you need to focus on specific compliance areas. This guide covers how to configure and manage CIS scan profiles.

## Prerequisites

- Rancher with CIS Benchmark app installed
- Cluster Owner or Cluster Admin permissions
- Understanding of CIS Kubernetes Benchmark checks
- `kubectl` access to the cluster

## Understanding CIS Scan Profiles

Rancher ships with several built-in scan profiles:

| Profile Name | Description |
|---|---|
| rke-cis-1.6 | Standard RKE1 benchmark |
| rke-cis-1.6-hardened | Hardened RKE1 benchmark |
| rke2-cis-1.6 | Standard RKE2 benchmark |
| rke2-cis-1.6-hardened | Hardened RKE2 benchmark |
| k3s-cis-1.6 | Standard K3s benchmark |
| k3s-cis-1.6-hardened | Hardened K3s benchmark |

## Step 1: View Existing Scan Profiles

```bash
# List all available scan profiles

kubectl get clusterscanprofile -A

# View details of a specific profile
kubectl describe clusterscanprofile rke2-cis-1.6 -n cattle-cis-system

# Get the profile YAML for review
kubectl get clusterscanprofile rke2-cis-1.6 \
  -n cattle-cis-system -o yaml
```

## Step 2: Create a Custom Scan Profile

Create a custom profile based on an existing one:

```yaml
# custom-cis-profile.yaml - Custom profile with specific checks skipped
apiVersion: cis.cattle.io/v1
kind: ClusterScanProfile
metadata:
  name: my-custom-profile
  namespace: cattle-cis-system
spec:
  # Base the custom profile on the hardened RKE2 profile
  benchmarkVersion: rke2-cis-1.6
  skipTests:
  # Skip checks that don't apply to your environment
  # Format: <section>.<check-number>

  # Skip this check if you're using a managed etcd service
  - "2.1"
  - "2.2"

  # Skip if your organization uses a different admission controller
  - "1.2.10"

  # Skip if using a cloud provider's IAM instead of RBAC
  - "3.1.1"

  # Skip checks requiring manual verification in your environment
  - "4.1.1"
  - "4.1.2"
```

```bash
kubectl apply -f custom-cis-profile.yaml

# Verify the profile was created
kubectl get clusterscanprofile my-custom-profile -n cattle-cis-system
```

## Step 3: Create a Profile for PCI DSS Compliance

Focus scanning on checks relevant to PCI DSS:

```yaml
# pci-dss-profile.yaml - Profile focused on PCI DSS relevant checks
apiVersion: cis.cattle.io/v1
kind: ClusterScanProfile
metadata:
  name: pci-dss-focused
  namespace: cattle-cis-system
spec:
  benchmarkVersion: rke2-cis-1.6
  skipTests:
  # Only run checks relevant to PCI DSS:
  # Keep network policy checks (required for PCI DSS segmentation)
  # Keep authentication checks (PCI DSS requires strong authentication)
  # Keep encryption checks (PCI DSS requires data encryption)
  # Skip checks that are PCI DSS irrelevant for your setup

  # Skip container image validation (managed separately)
  - "5.1.1"
  - "5.1.2"

  # Skip pod security policy checks (using pod security admission instead)
  - "5.2.1"
  - "5.2.2"
  - "5.2.3"
```

## Step 4: Create a Minimal Profile for Development

For development clusters where strict compliance is less critical:

```yaml
# dev-profile.yaml - Minimal profile for development environments
apiVersion: cis.cattle.io/v1
kind: ClusterScanProfile
metadata:
  name: development-minimal
  namespace: cattle-cis-system
spec:
  benchmarkVersion: rke2-cis-1.6
  skipTests:
  # Skip checks not applicable to development
  - "1.1.1"   # Master node config permissions (managed node)
  - "1.1.2"   # Master node config ownership (managed node)
  - "2.1"     # etcd configuration (embedded etcd)
  - "2.2"     # etcd ownership (embedded etcd)
  - "3.1.1"   # Client cert auth (not used in dev)
  - "4.1.1"   # Kubelet service file permissions (managed)
  - "4.1.2"   # Kubelet service file ownership (managed)
```

## Step 5: Use Custom Profile in a Scan

```bash
# Run a scan using the custom profile
kubectl apply -f - <<EOF
apiVersion: cis.cattle.io/v1
kind: ClusterScan
metadata:
  name: custom-profile-scan
spec:
  # Reference your custom profile
  scanProfileName: my-custom-profile
EOF

# Monitor the scan
kubectl get clusterscan custom-profile-scan -w
```

## Step 6: Update a Profile Based on Scan Results

After reviewing scan results, update your profile to exclude new false positives:

```bash
# Get the current profile configuration
kubectl get clusterscanprofile my-custom-profile \
  -n cattle-cis-system -o yaml > my-custom-profile.yaml

# Edit the profile to add additional skips
# Then apply the update
kubectl apply -f my-custom-profile.yaml
```

## Step 7: Profile Management Best Practices

```bash
# Document profile changes by adding annotations
kubectl annotate clusterscanprofile my-custom-profile \
  -n cattle-cis-system \
  "compliance.example.com/reason"="Skip 2.1-2.2: Using managed etcd via cloud provider" \
  "compliance.example.com/reviewer"="security-team" \
  "compliance.example.com/review-date"="2026-03-20"

# Create different profiles for different cluster types
# - production-hardened-profile: Strictest settings
# - staging-profile: Close to production but with some exceptions
# - development-profile: Permissive for development velocity
# - external-facing-profile: Extra strict for internet-facing clusters
```

## Conclusion

Custom CIS scan profiles allow you to tailor security benchmark scans to your organization's specific environment and requirements. By documenting the reason for each skipped check and getting approval from your security team, you maintain compliance while acknowledging the practical constraints of your infrastructure. Regularly reviewing and updating your profiles as your environment evolves ensures your scans remain relevant and meaningful.
