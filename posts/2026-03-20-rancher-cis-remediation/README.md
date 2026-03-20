# How to Remediate CIS Benchmark Failures in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, CIS, Security, Compliance, Remediation

Description: A practical guide to remediating common CIS Kubernetes benchmark failures found by Rancher's CIS scanning tool.

After running CIS scans in Rancher, the next critical step is remediating the failures. CIS benchmark failures represent real security gaps that increase your cluster's attack surface. This guide covers the most common CIS benchmark failures in RKE2 clusters and provides step-by-step remediation guidance.

## Prerequisites

- Rancher with CIS Benchmark scan results showing failures
- Cluster admin privileges
- `kubectl` and SSH access to cluster nodes
- Understanding of your cluster architecture

## Common CIS Benchmark Failure Categories

CIS failures typically fall into these categories:

1. **API Server configuration** (Section 1.2)
2. **etcd configuration** (Section 2)
3. **Control plane configuration** (Section 3)
4. **Worker node configuration** (Section 4)
5. **Kubernetes Policies** (Section 5)

## Remediating API Server Failures

### 1.2.1 - Anonymous Authentication Disabled

```bash
# Check if anonymous auth is enabled (should be disabled)
# For RKE2, edit the server configuration
sudo vi /etc/rancher/rke2/config.yaml
```

```yaml
# /etc/rancher/rke2/config.yaml - Add these API server arguments
kube-apiserver-arg:
  # Disable anonymous authentication
  - "anonymous-auth=false"
  # Enable audit logging
  - "audit-log-path=/var/log/kube-audit/audit.log"
  # Require authentication for kubelet
  - "kubelet-client-certificate=/var/lib/rancher/rke2/server/tls/client-kube-apiserver.crt"
  - "kubelet-client-key=/var/lib/rancher/rke2/server/tls/client-kube-apiserver.key"
```

```bash
# Restart RKE2 to apply changes
sudo systemctl restart rke2-server
```

### 1.2.2 - Basic Authentication File Not Present

```bash
# Verify that basic auth is not configured (should not exist)
# Check the API server configuration
sudo ps aux | grep kube-apiserver | grep "basic-auth-file"

# If found, remove the basic-auth-file argument from the API server config
sudo vi /etc/rancher/rke2/config.yaml
# Remove any basic-auth-file entries
```

## Remediating etcd Failures

### 2.1 - etcd Data Directory Permissions

```bash
# Check current permissions on etcd data directory
ls -la /var/lib/rancher/rke2/server/db/etcd

# Fix permissions - should be 700
sudo chmod 700 /var/lib/rancher/rke2/server/db/etcd

# Fix ownership - should be owned by etcd user
sudo chown -R etcd:etcd /var/lib/rancher/rke2/server/db/etcd

# Verify the fix
ls -la /var/lib/rancher/rke2/server/db/
```

## Remediating Kubelet Failures

### 4.2.1 - Anonymous Authentication Disabled on Kubelet

```yaml
# /etc/rancher/rke2/config.yaml - Kubelet configuration
kubelet-arg:
  # Disable anonymous authentication to the kubelet API
  - "anonymous-auth=false"
  # Require authorization for kubelet
  - "authorization-mode=Webhook"
  # Enable client certificate authentication
  - "client-ca-file=/var/lib/rancher/rke2/agent/client-ca.crt"
  # Protect kernel defaults
  - "protect-kernel-defaults=true"
  # Read-only port disabled (port 0 = disabled)
  - "read-only-port=0"
```

## Remediating Network Policy Failures

### 5.3.2 - All Namespaces Have Network Policies

```bash
# Check which namespaces don't have network policies
kubectl get networkpolicy -A

# Add a default deny-all policy to each namespace
for ns in $(kubectl get namespaces -o jsonpath='{.items[*].metadata.name}'); do
  # Skip system namespaces
  if [[ "$ns" != "kube-system" ]] && \
     [[ "$ns" != "kube-public" ]] && \
     [[ "$ns" != "cattle-system" ]]; then

    # Check if a deny-all policy already exists
    if ! kubectl get networkpolicy default-deny -n $ns 2>/dev/null; then
      echo "Adding default-deny policy to namespace: $ns"
      kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: $ns
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF
    fi
  fi
done
```

## Remediating RBAC Failures

### 5.1.3 - Minimize Wildcard Usage in Roles

```bash
# Find roles with wildcard permissions (security risk)
kubectl get clusterroles -o yaml | grep -B5 '"\*"' | grep "name:"

# Find namespace roles with wildcards
kubectl get roles -A -o yaml | grep -B5 '"\*"' | grep -E "name:|namespace:"

# Review and replace wildcard permissions with specific ones
# Instead of this:
cat << 'EOF'
# BAD: Wildcard permissions
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
EOF

# Use this:
cat << 'EOF'
# GOOD: Specific permissions
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
EOF
```

## Remediating Pod Security Failures

### 5.2.1 - Do Not Admit Privileged Containers

```yaml
# Apply Pod Security Standards using namespace labels
# For existing namespaces, use the pod-security admission controller

# Enforce restricted pod security standard
kubectl label namespace my-app \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/warn-version=latest

# For namespaces that need baseline (less strict)
kubectl label namespace monitoring \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/warn=restricted
```

## Step: Verify Remediations

```bash
# Run a new CIS scan after remediating issues
kubectl apply -f - <<EOF
apiVersion: cis.cattle.io/v1
kind: ClusterScan
metadata:
  name: post-remediation-scan
spec:
  scanProfileName: rke2-cis-1.6-profile-hardened
EOF

# Wait for the scan to complete
kubectl get clusterscan post-remediation-scan -w

# Compare results with the previous scan
# The number of failures should decrease
kubectl get clusterscan post-remediation-scan \
  -o jsonpath='{.status.summary}'
```

## Conclusion

Remediating CIS benchmark failures is an iterative process that requires collaboration between security, operations, and development teams. Start with the highest-severity failures and work systematically through each category. After each round of remediations, run a new CIS scan to verify progress. Remember to test remediations in a non-production environment first, as some changes (like disabling anonymous auth on the kubelet) can impact running workloads if not implemented carefully.
