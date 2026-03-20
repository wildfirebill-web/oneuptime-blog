# How to Set Up Rancher for Financial Services - For

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Financial Services, PCI-DSS, Compliance, Security, Kubernetes

Description: Configure Rancher for financial services workloads with PCI-DSS compliance controls, network isolation, audit logging, encryption, and the security posture required for payment processing and...

## Introduction

Financial services organizations running Kubernetes face unique challenges: PCI-DSS requirements for cardholder data environments, SOX audit controls, strict network segmentation between trading systems and customer-facing apps, and zero-tolerance for unplanned downtime. Rancher's multi-cluster management, CIS benchmark scanning, and RBAC capabilities make it well-suited for financial services, but require careful configuration.

## PCI-DSS Compliance Architecture

```text
                    ┌──────────────────────────────────┐
                    │     Rancher Management Cluster   │
                    │     (air-gapped, dedicated)      │
                    └──────────────┬───────────────────┘
                                   │
          ┌────────────────────────┼───────────────────────┐
          │                        │                       │
  ┌───────▼────────┐    ┌──────────▼──────┐    ┌──────────▼──────┐
  │  CDE Cluster   │    │  Non-CDE Prod   │    │  Non-Prod       │
  │  (PCI scope)   │    │  Cluster        │    │  Cluster        │
  │                │    │                 │    │                 │
  │ payment-ns     │    │ web-ns          │    │ dev/staging     │
  │ card-storage   │    │ customer-api    │    │                 │
  └────────────────┘    └─────────────────┘    └─────────────────┘
  Isolated network      Separate from CDE       Separate cluster
```

## Step 1: Harden the CDE Cluster

```yaml
# CDE cluster must have:

# 1. Dedicated nodes (no shared workloads)
# 2. etcd encryption at rest
# 3. Network isolation from other clusters
# 4. Strict audit logging

# RKE2 config for CDE cluster
# /etc/rancher/rke2/config.yaml
kube-apiserver-arg:
  - "encryption-provider-config=/etc/rancher/rke2/encryption-config.yaml"
  - "audit-log-path=/var/log/audit/kube-apiserver.log"
  - "audit-policy-file=/etc/rancher/rke2/audit-policy.yaml"
  - "audit-log-maxage=365"    # 1 year retention for PCI
  - "enable-admission-plugins=NodeRestriction,PodSecurity"

# Enforce restricted pod security
kube-apiserver-arg:
  - "admission-control-config-file=/etc/rancher/rke2/psa-config.yaml"
```

## Step 2: Network Segmentation

```yaml
# Strict network policies for CDE namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: cde-isolation
  namespace: payment-processing
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
  ingress:
    # Only allow from API gateway in same namespace
    - from:
        - podSelector:
            matchLabels:
              role: api-gateway
      ports:
        - port: 8443
  egress:
    # Only allow to approved payment processor IPs
    - to:
        - ipBlock:
            cidr: 203.0.113.0/24    # Payment processor CIDR
      ports:
        - port: 443
    # DNS
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - port: 53
          protocol: UDP
```

## Step 3: Secrets Management

```bash
# Never store PANs (card numbers) in Kubernetes Secrets
# Use HashiCorp Vault with Transit Secrets Engine for encryption

# Enable Vault Transit for tokenization
vault secrets enable transit
vault write -f transit/keys/payment-tokenizer type=aes256-gcm96

# Tokenize card data before storing
vault write transit/encrypt/payment-tokenizer \
  plaintext=$(echo -n "4111111111111111" | base64) \
  context=$(echo -n "customer-id" | base64)
```

## Step 4: TLS and Certificate Management

```bash
# Install cert-manager for automated certificate rotation
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --set installCRDs=true \
  --set prometheus.enabled=true

# Internal CA for service-to-service mTLS
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: internal-ca
spec:
  ca:
    secretName: internal-ca-key-pair
EOF
```

## Step 5: PCI-DSS Audit Logging

```bash
# Forward all audit logs to immutable SIEM
# Integrate with Splunk, IBM QRadar, or AWS Security Hub

kubectl apply -f - <<EOF
apiVersion: logging.banzaicloud.io/v1beta1
kind: ClusterFlow
metadata:
  name: audit-to-siem
  namespace: cattle-logging-system
spec:
  match:
    - select:
        labels:
          log-type: audit
  filters:
    - tag_normaliser: {}
  globalOutputRefs:
    - splunk-output
EOF
```

## Step 6: Run CIS Benchmarks

```bash
# Schedule monthly CIS benchmark scans
kubectl apply -f - <<EOF
apiVersion: cis.cattle.io/v1
kind: ClusterScan
metadata:
  name: monthly-pci-scan
spec:
  scanProfileName: rke2-cis-1.24-hardened-profile
EOF

# Review results and remediate findings
kubectl get clusterscans
kubectl describe clusterscansummary monthly-pci-scan
```

## PCI-DSS Control Mapping

| PCI Requirement | Rancher Control |
|---|---|
| Req 1: Network controls | Network Policies, Calico |
| Req 2: Secure configs | CIS Benchmark, Pod Security |
| Req 3: Protect cardholder data | etcd encryption, Vault |
| Req 8: Identify/authenticate | SSO/OIDC, RBAC |
| Req 10: Audit logging | Kubernetes audit log + SIEM |
| Req 11: Security testing | CIS scans, Trivy |

## Conclusion

Rancher provides the building blocks for PCI-DSS compliant Kubernetes deployments: network isolation via network policies, etcd encryption, CIS benchmark scanning, and centralized RBAC. The CDE cluster should be isolated from all other workloads, with strict network segmentation, encrypted storage, and immutable audit logs forwarded to a SIEM. Regular CIS benchmark scans and penetration testing validate the security posture against PCI-DSS requirements.
