# How to Set Up Rancher for Financial Services

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Financial-services, PCI-DSS, Kubernetes, Compliance, Security

Description: A step-by-step guide to configuring Rancher for PCI DSS-compliant financial services environments, covering network segmentation, access controls, and audit requirements.

## Overview

Financial services organizations running Kubernetes must comply with PCI DSS (Payment Card Industry Data Security Standard), SOX (Sarbanes-Oxley), and often GLBA (Gramm-Leach-Bliley Act). These regulations demand strict network segmentation, access controls, encryption, and detailed audit trails. This guide covers the key Rancher configurations for a compliant financial services deployment.

## Prerequisites

- Rancher v2.7+ with enterprise support (Rancher Prime recommended)
- RKE2 clusters with CIS profile
- Dedicated network segments for cardholder data environments (CDE)
- HSM or KMS for secrets management (Vault recommended)
- Certified scanning tools for PCI DSS scans

## Step 1: CDE Network Segmentation

PCI DSS requires strict isolation of the Cardholder Data Environment (CDE):

```yaml
# Namespace-level isolation using Rancher Projects

# Create a dedicated Project for CDE workloads with no shared namespaces

# NetworkPolicy: CDE namespace default deny
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: cde-default-deny
  namespace: cardholder-env
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
---
# Only allow PCI payment processor communication on specific port
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: cde-payment-processor
  namespace: cardholder-env
spec:
  podSelector:
    matchLabels:
      app: payment-service
  egress:
    - to:
        - ipBlock:
            cidr: 10.200.1.0/24  # Payment processor IP range
      ports:
        - protocol: TCP
          port: 443
```

## Step 2: Secrets Management with HashiCorp Vault

PCI DSS requires strong key management for cryptographic operations:

```bash
# Install Vault in Kubernetes
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install vault hashicorp/vault \
  --namespace vault \
  --create-namespace \
  --set server.ha.enabled=true \
  --set server.ha.replicas=3
```

```yaml
# Use Vault Agent Injector to inject secrets into pods
apiVersion: v1
kind: Pod
metadata:
  name: payment-processor
  annotations:
    # Vault injects the card encryption key automatically
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/role: "payment-service"
    vault.hashicorp.com/agent-inject-secret-card-key: "secret/pci/card-encryption-key"
spec:
  containers:
    - name: payment-processor
      image: payment-service:v1.0.0
```

## Step 3: CIS Hardened RKE2 Clusters

```yaml
# /etc/rancher/rke2/config.yaml
profile: cis-1.23
selinux: true
secrets-encryption: true
protect-kernel-defaults: true

# Audit policy for PCI DSS compliance
audit-policy-file: /etc/rancher/rke2/audit-policy.yaml
audit-log-path: /var/log/kubernetes/audit.log
audit-log-maxsize: 100
audit-log-maxbackup: 10
audit-log-maxage: 365    # 1 year retention for PCI DSS
```

## Step 4: PCI DSS Audit Policy

```yaml
# Comprehensive audit policy for PCI DSS
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Log all access to Secrets (card data credentials)
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["secrets"]
  # Log all Pod exec (Requirement 8.2 - no shared accounts)
  - level: RequestResponse
    verbs: ["create"]
    resources:
      - group: ""
        resources: ["pods/exec"]
  # Log all RBAC changes (Requirement 7 - access control)
  - level: RequestResponse
    resources:
      - group: "rbac.authorization.k8s.io"
        resources: ["roles", "rolebindings", "clusterroles", "clusterrolebindings"]
  - level: Metadata
    omitStages:
      - RequestReceived
```

## Step 5: Multi-Factor Authentication

PCI DSS Requirement 8.4 mandates MFA for all access to the CDE:

```text
Rancher UI → Global Settings → Auth Configuration → SAML
- Configure with your corporate SSO (Okta, Azure AD) with MFA enforcement
- Require MFA for all users accessing CDE project clusters
- Set session timeout: 15 minutes for CDE access
```

## Step 6: Container Image Security

PCI DSS Requirement 6 mandates secure application development:

```yaml
# Kubewarden policy: Block containers from public registries
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: restrict-public-registry
spec:
  module: registry://ghcr.io/kubewarden/policies/allowed-image-repositories:v0.1.0
  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations: ["CREATE"]
  settings:
    allowedRegistries:
      - "registry.internal.bank.com"   # Internal registry only
```

## Step 7: NeuVector for PCI DSS Runtime Security

```bash
# Enable NeuVector with PCI DSS compliance mode
# In NeuVector UI:
# Settings → Compliance → Enable PCI DSS Profile
# This automatically scans for PCI DSS control violations
```

## Step 8: Vulnerability Scanning and Patch Management

PCI DSS Requirement 6.3 requires vulnerability scanning:

```yaml
# Schedule regular NeuVector scans
# Configure in NeuVector UI: Vulnerabilities → Scan Schedule
# Or via API:
curl -X POST https://neuvector:10443/v1/scan/registry \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "config": {
      "name": "pci-registry-scan",
      "registry": "registry.internal.bank.com",
      "schedule": {"interval": 86400}
    }
  }'
```

## Step 9: Rancher CIS Scanning

Run regular CIS benchmark scans on all clusters:

```yaml
# ClusterScan with PCI-relevant profile
apiVersion: cis.cattle.io/v1
kind: ClusterScan
metadata:
  name: pci-cis-scan
  namespace: cis-operator-system
spec:
  scanProfileName: rke2-cis-1.23-hardened
  cronSchedule: "0 2 * * 0"  # Weekly Sunday 2 AM
```

## Compliance Checklist (PCI DSS)

- [ ] CDE namespace isolated with default-deny NetworkPolicies
- [ ] Secrets encrypted at rest (etcd encryption + Vault)
- [ ] RKE2 CIS hardened profile active on all CDE clusters
- [ ] Audit logs retained 12 months (1 year online, 3 months accessible)
- [ ] MFA enforced via SSO for all admin access
- [ ] Container images from approved registries only
- [ ] Weekly vulnerability scans on container images
- [ ] NeuVector PCI DSS compliance mode enabled
- [ ] Regular CIS benchmark scans scheduled
- [ ] RBAC following least-privilege principle

## Conclusion

Configuring Rancher for financial services requires careful alignment with PCI DSS requirements across network segmentation, access control, encryption, and audit logging. RKE2's built-in CIS hardening, NeuVector's runtime security, and Longhorn's encrypted storage provide a strong foundation. Supplement with HashiCorp Vault for secrets management and a comprehensive audit log retention strategy to meet PCI DSS requirements. Regular scanning and compliance reporting help maintain ongoing compliance.
