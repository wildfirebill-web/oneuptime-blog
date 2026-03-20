# How to Set Up Rancher for Government and FedRAMP

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: rancher, government, fedramp, fips, kubernetes, compliance

Description: A comprehensive guide to configuring Rancher for US government FedRAMP compliance, covering FIPS 140-2, STIG hardening, and FedRAMP High authorization requirements.

## Overview

US government agencies and contractors handling federal data must comply with FedRAMP (Federal Risk and Authorization Management Program). FedRAMP authorization requires FIPS 140-2 cryptographic modules, NIST 800-53 security controls, and STIG (Security Technical Implementation Guide) hardening. RKE2 with FIPS mode and Rancher provide a strong foundation for FedRAMP-compliant Kubernetes deployments.

## Prerequisites

- RHEL 8/9 or Rocky Linux 8/9 with FIPS mode enabled at OS level
- RKE2 v1.24+ with FIPS 140-2 build
- DoD-approved PKI certificates (or FedRAMP-authorized CA)
- SIEM integration for log forwarding
- DISA STIG viewer tools

## Step 1: Enable FIPS at OS Level

FIPS must be enabled at the OS level before installing RKE2:

```bash
# Enable FIPS on RHEL/Rocky Linux
fips-mode-setup --enable
reboot

# Verify FIPS is enabled
fips-mode-setup --check
# Output: FIPS mode is enabled

# Verify crypto policy
update-crypto-policies --show
# Should show: FIPS
```

## Step 2: Install RKE2 with FIPS Mode

```bash
# Install RKE2 FIPS build
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="server" sh -

# Create FIPS-enabled config
cat > /etc/rancher/rke2/config.yaml << 'EOF'
# Enable FIPS 140-2 compliance
fips: true

# CIS hardening profile
profile: cis-1.23

# STIG-required settings
selinux: true
secrets-encryption: true
protect-kernel-defaults: true

# Audit logging (NIST 800-53 AU controls)
audit-policy-file: /etc/rancher/rke2/audit-policy.yaml
audit-log-path: /var/log/kubernetes/audit.log
audit-log-maxage: 365

# TLS configuration (DoD PKI)
tls-san:
  - "k8s.agency.gov"
  - "10.0.1.100"
EOF

systemctl enable --now rke2-server
```

## Step 3: STIG Hardening

The DISA Kubernetes STIG requires specific configurations:

```yaml
# Pod Security Admission for privileged access restriction (STIG V-242395)
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
  - name: PodSecurity
    configuration:
      apiVersion: pod-security.admission.config.k8s.io/v1
      kind: PodSecurityConfiguration
      defaults:
        enforce: "restricted"
        enforce-version: "latest"
      exemptions:
        namespaces:
          - kube-system
          - cattle-system
```

```bash
# Verify STIG controls with kube-bench
kubectl run kube-bench \
  --image=aquasec/kube-bench:latest \
  --restart=Never \
  -- run --targets node \
  --benchmark rke2-cis-1.23
```

## Step 4: DoD PKI Certificate Configuration

```yaml
# RKE2 TLS config with DoD PKI
# /etc/rancher/rke2/config.yaml
tls-san:
  - "k8s.agency.gov"

# Mount DoD root CA certificates
# Copy DoD root CA to:
# /etc/rancher/rke2/server/tls/
```

## Step 5: FedRAMP Audit Logging (NIST AU Controls)

```yaml
# Comprehensive audit policy meeting NIST 800-53 AU-2, AU-3, AU-12
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # AU-9: Protect audit information
  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]
    resources:
      - group: ""
        resources: ["endpoints", "services", "services/status"]
  # Log all authentication events
  - level: Metadata
    stages:
      - ResponseStarted
    nonResourceURLs:
      - /api*
      - /version
  # Log all resource modifications at RequestResponse level
  - level: RequestResponse
    verbs: ["create", "update", "patch", "delete", "deletecollection"]
  # Log all reads at Metadata level
  - level: Metadata
    verbs: ["get", "list", "watch"]
    omitStages:
      - RequestReceived
```

## Step 6: SIEM Integration

FedRAMP requires centralized log management. Forward audit logs to your SIEM:

```yaml
# Deploy Fluentd/Fluent Bit for log forwarding
# Install via Rancher Logging
helm install rancher-logging rancher-charts/rancher-logging \
  --namespace cattle-logging-system \
  --create-namespace
```

```yaml
# ClusterOutput to Splunk SIEM
apiVersion: logging.banzaicloud.io/v1beta1
kind: ClusterOutput
metadata:
  name: splunk-output
  namespace: cattle-logging-system
spec:
  splunkHec:
    hec_host: splunk.agency.gov
    hec_port: 8088
    hec_token: ${HEC_TOKEN}
    ca_file: /etc/ssl/dod-root-ca.pem
    insecure_ssl: false
```

## Step 7: Identity and Access Management

FedRAMP requires PIV/CAC card authentication:

```
Rancher UI → Global Settings → Auth Configuration → SAML
- Configure SAML with DoD ADFS
- Enforce PIV/CAC card authentication via ADFS
- Set session timeout: 15 minutes (AC-12 control)
- Require re-authentication for privilege escalation
```

## Step 8: Continuous Monitoring (ConMon)

FedRAMP ConMon requires regular vulnerability scanning and compliance reporting:

```bash
# Schedule weekly CIS scans
# In Rancher UI: Cluster → CIS Scans → Schedule Scan
# Profile: rke2-cis-1.23-hardened
# Schedule: Weekly

# NeuVector FedRAMP compliance profile
# In NeuVector UI: Policy → Compliance → Run Compliance Scan
# Select: NIST, CIS Docker, CIS Kubernetes
```

## Step 9: Incident Response

Configure NeuVector to automatically respond to security events:

```yaml
# NeuVector Group rule for automatic quarantine on suspicious process
# Settings → Response Rules:
# - Trigger: Suspicious Process
# - Action: Quarantine
# - Target: All pods in CDE namespace
```

## FedRAMP Control Mapping

| FedRAMP Control | Rancher Implementation |
|---|---|
| AC-2 (Account Management) | RBAC + LDAP/SAML integration |
| AC-17 (Remote Access) | TLS-only access, MFA enforced |
| AU-2 (Audit Events) | Kubernetes audit logging |
| AU-9 (Audit Log Protection) | Log forwarding to SIEM |
| CM-6 (Configuration Settings) | RKE2 CIS profile, STIG |
| IA-2 (MFA) | PIV/CAC via SAML/ADFS |
| SC-8 (Transmission Confidentiality) | TLS 1.2+ everywhere |
| SC-28 (Protection at Rest) | etcd encryption, Longhorn encryption |
| SI-2 (Flaw Remediation) | Regular image scanning, patching |

## Conclusion

Achieving FedRAMP compliance with Rancher requires FIPS 140-2 enabled RKE2, STIG hardening, comprehensive audit logging, PIV/CAC authentication, and continuous monitoring. The SUSE Rancher stack — including RKE2, NeuVector, and Longhorn — provides most of the required controls out of the box. Plan to engage a FedRAMP 3PAO (Third Party Assessment Organization) for your official authorization and review. Maintain your Plan of Action and Milestones (POA&M) and conduct monthly continuous monitoring to preserve your authorization.
