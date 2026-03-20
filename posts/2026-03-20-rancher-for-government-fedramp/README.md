# How to Set Up Rancher for Government and FedRAMP

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Government, FedRAMP, Compliance, Air-Gapped, Security, Kubernetes

Description: Configure Rancher for US government workloads meeting FedRAMP requirements including air-gapped deployments, FIPS 140-2 encryption, STIG compliance, audit logging, and the security controls...

## Introduction

US government workloads in Kubernetes must comply with FedRAMP (for cloud) or DISA STIGs (for on-premises). Both frameworks require FIPS 140-2 validated cryptography, comprehensive audit logging, strict access controls, and often air-gapped deployments without internet connectivity. RKE2 is FIPS 140-2 compliant when configured correctly, making it the recommended distribution for government Kubernetes.

## FedRAMP Architecture

```text
Air-Gapped Environment
┌─────────────────────────────────────────────────────┐
│                                                     │
│  ┌─────────────────┐    ┌──────────────────────┐   │
│  │  Private Harbor │    │   Rancher Management  │   │
│  │  Registry       │    │   (FIPS enabled)      │   │
│  └─────────────────┘    └──────────────────────┘   │
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │         Government Production Cluster       │   │
│  │         (RKE2 + FIPS 140-2)                 │   │
│  └─────────────────────────────────────────────┘   │
│                                                     │
└─────────────────────────────────────────────────────┘
        │ Approval gate
  Outside networks
```

## Step 1: Enable FIPS 140-2 on RKE2

```bash
# Install RKE2 with FIPS mode

# FIPS mode uses NIST-approved cryptographic algorithms only

# Download FIPS-compliant RKE2 binary
curl -sfL https://get.rke2.io | INSTALL_RKE2_FIPS=true sh -

# Verify FIPS mode is active
rke2 --version | grep -i fips

# Configure FIPS-compliant TLS
cat > /etc/rancher/rke2/config.yaml << 'EOF'
# FIPS-compliant cipher suites only
kube-apiserver-arg:
  - "tls-cipher-suites=TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384"
  - "tls-min-version=VersionTLS12"
EOF
```

## Step 2: Air-Gapped Rancher Installation

```bash
# Download all required images for air-gapped install
# On an internet-connected machine:
helm pull rancher/rancher --version 2.8.0 --untar

# Download Rancher image list
curl -sfL https://github.com/rancher/rancher/releases/download/v2.8.0/rancher-images.txt -o rancher-images.txt

# Save images to tarball
cat rancher-images.txt | xargs docker pull
docker save $(cat rancher-images.txt | tr '\n' ' ') | gzip > rancher-images.tar.gz

# Transfer to air-gapped environment and load
docker load -i rancher-images.tar.gz

# Push to private Harbor registry
cat rancher-images.txt | while read image; do
  new_tag="harbor.gov.internal/rancher/${image##*/}"
  docker tag "$image" "$new_tag"
  docker push "$new_tag"
done
```

## Step 3: Apply DISA STIG Controls

```bash
# Enable STIGatrix or apply STIG hardening scripts
# Key STIG controls for Kubernetes (K8S STIG V1R14):

# V-242376: Enable audit logging
kube-apiserver-arg:
  - "audit-log-path=/var/log/kubernetes/audit.log"
  - "audit-log-maxage=365"
  - "audit-log-maxsize=100"

# V-242381: Disable anonymous auth
kube-apiserver-arg:
  - "anonymous-auth=false"

# V-242384: Enable Node restriction
kube-apiserver-arg:
  - "enable-admission-plugins=NodeRestriction"

# V-242386: Disable profiling
kube-apiserver-arg:
  - "profiling=false"
```

## Step 4: Continuous Compliance Monitoring

```bash
# Run OpenSCAP scans for STIG compliance
# Install on RHEL nodes
yum install -y scap-security-guide openscap-scanner

# Run Kubernetes STIG check
oscap xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_stig \
  --results /var/log/oscap-results.xml \
  /usr/share/xml/scap/ssg/content/ssg-rhel8-ds.xml

# Integrate with Rancher CIS Benchmark
kubectl apply -f - <<EOF
apiVersion: cis.cattle.io/v1
kind: ClusterScanBenchmark
metadata:
  name: dod-hardened
spec:
  clusterProvider: rke2
  minKubernetesVersion: "1.24.0"
  customBenchmarkConfigMapName: dod-k8s-stig-benchmark
EOF
```

## Step 5: CAC/PIV Authentication

```bash
# Configure Rancher to use CAC/PIV via SAML
# Connect to AD FS or Okta (which supports CAC/PIV)

# In Rancher UI:
# Global Settings > Authentication > SAML > Microsoft AD FS

# AD FS configuration:
# Entity ID: https://rancher.gov.internal
# Assertion Consumer Service URL: https://rancher.gov.internal/v1-saml/adfs/saml/acs

# Required claims:
# - Email
# - Name
# - UPN (for group mapping)
```

## Step 6: Immutable Audit Log Forwarding

```yaml
# Forward audit logs to SIEM (Splunk, ArcSight)
apiVersion: logging.banzaicloud.io/v1beta1
kind: ClusterOutput
metadata:
  name: siem-output
  namespace: cattle-logging-system
spec:
  forward:
    servers:
      - host: siem.gov.internal
        port: 514
        weight: 100
    transport: tls
    tls_cert_path: /etc/ssl/certs/gov-ca.crt
    tls_client_cert_path: /etc/ssl/certs/cluster-client.crt
    tls_client_private_key_path: /etc/ssl/private/cluster-client.key
```

## FedRAMP Control Summary

| NIST 800-53 Control | Implementation |
|---|---|
| AC-2 Account Management | Rancher RBAC + AD/LDAP |
| AU-2 Audit Events | Kubernetes audit log |
| IA-2 MFA | CAC/PIV via SAML |
| SC-28 At-Rest Protection | FIPS etcd encryption |
| SI-3 Malware Protection | Trivy + Falco |
| CM-6 Configuration Settings | CIS/STIG benchmarks |

## Conclusion

RKE2 with FIPS mode enabled provides a solid foundation for FedRAMP and DISA STIG compliance. The key requirements are: FIPS 140-2 cryptography throughout, air-gapped deployment with a private registry, CAC/PIV authentication via AD FS SAML integration, comprehensive audit logging forwarded to an approved SIEM, and regular STIG compliance scanning. SUSE offers Rancher Government Solutions (RGS) with pre-validated compliance configurations for federal customers.
