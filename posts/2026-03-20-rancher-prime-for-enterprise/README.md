# How to Set Up Rancher Prime for Enterprise

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Rancher Prime, Enterprise, SUSE, Support, Compliance, Security

Description: Set up Rancher Prime for enterprise deployments with SUSE commercial support, enhanced security features, extended lifecycle support, and the additional enterprise capabilities that distinguish Prime from open-source Rancher.

## Introduction

Rancher Prime is SUSE's commercial offering of Rancher, providing enterprise-grade support, extended lifecycle management, additional security features, and compliance tooling. For organizations running Kubernetes at scale with SLA requirements, Rancher Prime adds the enterprise support layer and additional capabilities needed for production deployments in regulated industries.

## Rancher Prime vs. Open Source Rancher

| Feature | Rancher (OSS) | Rancher Prime |
|---|---|---|
| Support | Community | 24/7 SUSE Enterprise Support |
| Lifecycle | Standard | Extended (30 months) |
| Security patches | Community | Priority CVE patching |
| CIS Hardened images | No | Yes |
| FIPS 140-2 images | No | Yes |
| Air-gap support | Basic | Full enterprise air-gap |
| Compliance reports | Basic | Enhanced |
| SLA | None | 4-hour critical response |

## Step 1: Obtain Rancher Prime License

```bash
# Register at https://www.suse.com/products/rancher/
# Download your license key from SUSE Customer Center (SCC)

# Configure SUSE registry credentials
kubectl create secret docker-registry suse-registry-credentials \
  --docker-server=registry.suse.com \
  --docker-username=<scc-username> \
  --docker-password=<scc-password> \
  -n cattle-system
```

## Step 2: Install Rancher Prime

```bash
# Add SUSE Rancher Prime Helm chart repository
helm repo add rancher-prime https://charts.rancher.com/server-charts/prime
helm repo update

# Install Rancher Prime with enterprise configuration
helm install rancher rancher-prime/rancher \
  --namespace cattle-system \
  --create-namespace \
  --set hostname=rancher.company.com \
  --set bootstrapPassword=<bootstrap-password> \
  --set ingress.tls.source=letsEncrypt \
  --set letsEncrypt.email=platform@company.com \
  --set global.cattle.psp.enabled=false \
  --set replicas=3 \
  --version 2.8.0
```

## Step 3: Configure Extended Support Lifecycle

```bash
# Enable SUSE Manager integration for lifecycle management
# SUSE Manager provides:
# - Automated patch management
# - Lifecycle notifications
# - Compliance baselines

# Register cluster with SUSE Manager
curl -o /usr/share/rhn/RHN-ORG-TRUSTED-SSL-CERT \
  https://suma.company.com/pub/RHN-ORG-TRUSTED-SSL-CERT

rhnreg_ks \
  --serverUrl=https://suma.company.com/XMLRPC \
  --activationkey=<activation-key>
```

## Step 4: Enable CIS Hardened Images

```yaml
# Use SUSE's CIS-hardened container images for system components
# These images follow CIS Docker Benchmark Level 2

# Rancher Prime ships with hardened component images:
# - rancher/hardened-kubernetes
# - rancher/hardened-etcd
# - rancher/hardened-flannel
# - rancher/hardened-calico

# Configure RKE2 to use hardened images
apiVersion: provisioning.cattle.io/v1
kind: Cluster
metadata:
  name: production-hardened
spec:
  rkeConfig:
    # Using hardened system images
    systemImages:
      kubernetes: rancher/hardened-kubernetes:v1.28.6-rke2r1-build20240116
      etcd: rancher/hardened-etcd:v3.5.9-k3s1-build20231214
    machineGlobalConfig:
      # CIS profile
      profile: cis-1.23
      protect-kernel-defaults: true
      pod-security-admission-config-file: /etc/rancher/rke2/psa-config.yaml
```

## Step 5: Configure Enterprise Audit and Reporting

```bash
# Rancher Prime includes enhanced audit capabilities

# Enable comprehensive audit logging
cat >> /etc/rancher/rke2/config.yaml << 'EOF'
kube-apiserver-arg:
  - "audit-log-path=/var/log/rancher-audit/audit.log"
  - "audit-log-maxage=90"         # 90-day retention
  - "audit-policy-file=/etc/rancher/rke2/audit-policy.yaml"
  - "audit-log-format=json"
EOF

# Configure Rancher audit log
kubectl patch configmap rancher-config \
  -n cattle-system \
  --type merge \
  -p '{"data":{"auditLog.level": "3"}}'
```

## Step 6: SUSE Enterprise Support Integration

```bash
# Generate support bundle for SUSE support cases
# In Rancher UI: Help > Get Support > Generate Support Bundle

# Or via CLI
kubectl rancher-support-bundle collect \
  --name "support-case-123456" \
  --output /tmp/support-bundle.zip

# Submit to SUSE support portal
# https://scc.suse.com/support/requests

# Monitor SLAs via SUSE Customer Center:
# - Severity 1 (Critical): 4-hour response
# - Severity 2 (High): 8-hour response
# - Severity 3 (Medium): 2 business day response
# - Severity 4 (Low): 5 business day response
```

## Step 7: Rancher Prime Security Features

```bash
# NeuVector integration (included with Prime)
helm install neuvector neuvector/core \
  --namespace neuvector \
  --create-namespace \
  --set tag=5.3.0 \
  --set controller.image.repository=registry.suse.com/rancher/mirrored-neuvector-controller \
  --set manager.image.repository=registry.suse.com/rancher/mirrored-neuvector-manager \
  --set enforcer.image.repository=registry.suse.com/rancher/mirrored-neuvector-enforcer

# NeuVector provides:
# - Runtime security and zero-trust network segmentation
# - Container scanning
# - Compliance assessment (PCI, HIPAA, NIST)
# - DLP and WAF for containers
```

## Enterprise Deployment Checklist

- Rancher Prime license registered and activated
- SUSE registry credentials configured for hardened images
- CIS hardened image profile applied to all clusters
- Extended lifecycle support configured (30 months)
- SUSE Manager integrated for patch management
- 24/7 support contact configured in SUSE Customer Center
- NeuVector deployed for runtime security
- Audit log retention set to 90 days minimum
- Support bundle generation tested before incident

## Conclusion

Rancher Prime extends the open-source Rancher platform with enterprise support SLAs, CIS-hardened images, FIPS 140-2 compliance, and NeuVector security. For organizations with compliance requirements (PCI-DSS, HIPAA, FedRAMP) or SLA commitments to business stakeholders, Rancher Prime provides the commercial backing needed to run Kubernetes confidently at scale. The 24/7 SUSE support and extended 30-month lifecycle significantly reduce the operational burden of keeping enterprise Kubernetes up-to-date and secure.
