# How to Configure STIG Compliance in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, STIG, Security, Compliance, DoD

Description: Learn how to configure Rancher-managed Kubernetes clusters to meet DISA STIG (Security Technical Implementation Guide) compliance requirements.

DISA STIGs (Security Technical Implementation Guides) provide prescriptive security configuration guidance for Department of Defense (DoD) systems. For Kubernetes clusters in government or defense environments, STIG compliance is mandatory. This guide covers how to configure Rancher and RKE2 clusters to meet STIG requirements.

## Prerequisites

- Rancher v2.6+ with RKE2 clusters
- Root/admin access to cluster nodes
- STIG Viewer or similar tool for viewing STIG requirements
- Understanding of Kubernetes security concepts

## Understanding Kubernetes STIG Requirements

The DISA Kubernetes STIG (STIG ID: K8S) covers:

- **Authentication and Authorization**: Disable anonymous auth, require certificates
- **Audit Logging**: Comprehensive audit log configuration
- **Network Security**: Network policies, TLS requirements
- **Container Security**: Privileged container restrictions
- **Secrets Management**: Encryption at rest for secrets
- **RBAC**: Least privilege access controls

## Step 1: Configure API Server for STIG Compliance

```yaml
# /etc/rancher/rke2/config.yaml - API server STIG settings
kube-apiserver-arg:
  # STIG V-242390: Disable anonymous authentication
  - "anonymous-auth=false"

  # STIG V-242391: Enable audit logging
  - "audit-log-path=/var/log/kubernetes/audit.log"
  - "audit-log-maxage=30"
  - "audit-log-maxbackup=10"
  - "audit-log-maxsize=100"

  # STIG V-242400: Require client certificates for the scheduler
  - "requestheader-client-ca-file=/var/lib/rancher/rke2/server/tls/request-header-ca.crt"

  # STIG V-242402: Disable AlwaysAdmit admission controller
  - "enable-admission-plugins=NodeRestriction,PodSecurityAdmission"

  # STIG V-242403: Disable AlwaysPullImages not required as admission plugin handles this
  # STIG V-242435: Enable encryption at rest
  - "encryption-provider-config=/etc/kubernetes/encryption-config.yaml"

  # STIG V-245541: Set TLS minimum version
  - "tls-min-version=VersionTLS12"
  - "tls-cipher-suites=TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384"
```

## Step 2: Configure Encryption at Rest

```yaml
# /etc/kubernetes/encryption-config.yaml - Encrypt secrets at rest
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
    - secrets
    providers:
    # AES-CBC encryption (STIG requirement)
    - aescbc:
        keys:
        - name: key1
          # Generate with: head -c 32 /dev/urandom | base64
          secret: <BASE64_ENCODED_32_BYTE_KEY>
    # Identity is the fallback (unencrypted) - needed for reading existing secrets
    - identity: {}
```

```bash
# Generate an encryption key
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
echo "Encryption key: $ENCRYPTION_KEY"

# Create the encryption config
sudo mkdir -p /etc/kubernetes
sudo cat > /etc/kubernetes/encryption-config.yaml << EOF
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
    - secrets
    providers:
    - aescbc:
        keys:
        - name: key1
          secret: ${ENCRYPTION_KEY}
    - identity: {}
EOF

# Verify existing secrets are re-encrypted after enabling
kubectl get secrets -A -o json | kubectl replace -f -
```

## Step 3: Configure Audit Logging

```yaml
# /etc/kubernetes/audit-policy.yaml - Comprehensive audit policy
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # STIG V-242402: Log all privileged operations
  - level: RequestResponse
    userGroups: ["system:masters"]

  # Log all authentication failures
  - level: RequestResponse
    verbs: ["create", "update", "patch", "delete"]
    resources:
    - group: ""
      resources: ["secrets", "configmaps"]

  # Log pod security violations
  - level: RequestResponse
    resources:
    - group: ""
      resources: ["pods"]

  # Log RBAC changes
  - level: RequestResponse
    resources:
    - group: "rbac.authorization.k8s.io"
      resources: ["clusterroles", "clusterrolebindings", "roles", "rolebindings"]

  # Log everything else at metadata level
  - level: Metadata
    omitStages:
    - RequestReceived
```

## Step 4: Configure Kubelet for STIG Compliance

```yaml
# /etc/rancher/rke2/config.yaml - Kubelet STIG settings
kubelet-arg:
  # STIG V-242415: Disable anonymous authentication
  - "anonymous-auth=false"

  # STIG V-242416: Enable Webhook authorization
  - "authorization-mode=Webhook"

  # STIG V-242417: Enable client CA authentication
  - "client-ca-file=/var/lib/rancher/rke2/agent/client-ca.crt"

  # STIG V-242418: Disable read-only port
  - "read-only-port=0"

  # STIG V-242419: Protect kernel defaults
  - "protect-kernel-defaults=true"

  # STIG V-242420: Disable streaming connection idle timeout
  - "streaming-connection-idle-timeout=5m"

  # STIG V-242421: Set event record QPS
  - "event-qps=0"

  # Enable certificate rotation
  - "rotate-certificates=true"

  # Set TLS minimum version
  - "tls-min-version=VersionTLS12"
```

## Step 5: Configure Network Policies for STIG

```bash
# Apply default deny network policies to all namespaces
for ns in $(kubectl get namespaces -o name | cut -d/ -f2); do
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
done
```

## Step 6: Configure RBAC for STIG Compliance

```bash
# Identify and remediate over-privileged service accounts
# Check for service accounts with cluster-admin binding
kubectl get clusterrolebindings -o json | \
  python3 -c "
import json, sys
data = json.load(sys.stdin)
for item in data['items']:
    if item.get('roleRef', {}).get('name') == 'cluster-admin':
        subjects = item.get('subjects', [])
        for s in subjects:
            if s.get('kind') == 'ServiceAccount':
                print(f\"SA with cluster-admin: {s.get('namespace')}/{s.get('name')}\")
"

# Remove unnecessary cluster-admin bindings
# Review each service account and replace with least-privilege role
```

## Step 7: Verify STIG Compliance

```bash
# Run CIS hardened profile (closest available to STIG checks)
kubectl apply -f - <<EOF
apiVersion: cis.cattle.io/v1
kind: ClusterScan
metadata:
  name: stig-verification-scan
spec:
  scanProfileName: rke2-cis-1.6-profile-hardened
EOF

# Check for any remaining failures
kubectl get clusterscan stig-verification-scan \
  -o jsonpath='{.status.summary}'

# Also check with OpenSCAP if available on nodes
# oscap-docker kubernetes stig scan
```

## Conclusion

Achieving STIG compliance for Kubernetes in Rancher requires a comprehensive approach covering API server hardening, encryption at rest, audit logging, kubelet security, network policies, and RBAC. While Rancher's CIS scanning provides a good baseline, full STIG compliance may require additional configuration steps and manual verification. Always work with your security team and ISSM (Information System Security Manager) when implementing STIG requirements in DoD environments.
