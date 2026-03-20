# How to Configure K3s for FIPS 140-2 Compliance

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: K3s, Kubernetes, Security, FIPS, Compliance, Government

Description: Learn how to configure K3s to use FIPS 140-2 validated cryptographic modules for government and regulated industry deployments.

## Introduction

FIPS 140-2 (Federal Information Processing Standard 140-2) is a US government security standard for cryptographic modules. Organizations in government, defense, finance, and healthcare often require FIPS-validated cryptography. K3s supports FIPS compliance through the use of FIPS-validated Go cryptographic binaries. This guide explains how to configure K3s for FIPS 140-2 compliant operation.

## Prerequisites

- A FIPS-enabled Linux operating system (RHEL 8/9, Ubuntu with FIPS kernel, or similar)
- Kernel with FIPS mode enabled
- Understanding of FIPS requirements for your organization

## Step 1: Enable FIPS Mode on the Host OS

Before configuring K3s, the underlying OS must be in FIPS mode:

### RHEL/CentOS 8/9

```bash
# Install FIPS modules

dnf install -y crypto-policies-scripts

# Enable FIPS mode (requires reboot)
fips-mode-setup --enable

# Reboot to activate FIPS
reboot

# After reboot, verify FIPS is enabled
fips-mode-setup --check
# Output: FIPS mode is enabled.

# Verify kernel parameter
cat /proc/sys/crypto/fips_enabled
# Should output: 1
```

### Ubuntu 20.04/22.04

```bash
# Install Ubuntu FIPS modules (requires Ubuntu Pro subscription)
ua enable fips

# Or for FIPS-updates channel
ua enable fips-updates

# Reboot
reboot

# Verify
cat /proc/sys/crypto/fips_enabled
```

## Step 2: Install K3s FIPS Binary

K3s provides a FIPS-validated binary built with a FIPS 140-2 validated Go toolchain:

```bash
# Download the FIPS-compliant K3s binary
# The FIPS binary uses the BoringCrypto validated cryptographic library
INSTALL_K3S_VERSION="v1.29.3+k3s1"

# Download the FIPS binary directly
curl -Lo /usr/local/bin/k3s \
  "https://github.com/k3s-io/k3s/releases/download/${INSTALL_K3S_VERSION}/k3s-arm64"
# For AMD64:
# https://github.com/k3s-io/k3s/releases/download/${INSTALL_K3S_VERSION}/k3s

# Make executable
chmod +x /usr/local/bin/k3s

# For official FIPS builds, check the K3s GitHub for FIPS-specific releases
# or use the Rancher Government Solutions builds
```

### Using Rancher Government Solutions

For certified FIPS builds:

```bash
# Rancher Government provides FIPS-validated K3s builds
# Install script for FIPS-compliant version
curl -sfL https://get.k3s.io | \
  INSTALL_K3S_VERSION=v1.29.3+k3s1 \
  sh -

# Verify the binary uses FIPS-validated crypto
# The binary should be linked against BoringSSL/BoringCrypto
```

## Step 3: Configure K3s with FIPS-Compliant Settings

```yaml
# /etc/rancher/k3s/config.yaml
# FIPS-compliant K3s configuration

# Use only FIPS-approved TLS cipher suites
kube-apiserver-arg:
  # Restrict TLS to FIPS-approved versions
  - "tls-min-version=VersionTLS12"
  # FIPS-approved cipher suites only
  - "tls-cipher-suites=TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384"

kube-controller-manager-arg:
  - "tls-min-version=VersionTLS12"
  - "tls-cipher-suites=TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384"

kube-scheduler-arg:
  - "tls-min-version=VersionTLS12"
  - "tls-cipher-suites=TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256"

kubelet-arg:
  - "tls-min-version=VersionTLS12"
  - "tls-cipher-suites=TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384"
```

## Step 4: Enable Secrets Encryption

FIPS deployments should always encrypt etcd secrets:

```yaml
# /etc/rancher/k3s/config.yaml
secrets-encryption: true
```

This enables encryption of Kubernetes secrets at rest using AES-256.

## Step 5: Configure Audit Logging

FIPS deployments typically require comprehensive audit logging:

```yaml
# /etc/rancher/k3s/config.yaml
kube-apiserver-arg:
  - "audit-log-path=/var/log/k3s/audit.log"
  - "audit-log-maxage=30"
  - "audit-log-maxbackup=10"
  - "audit-log-maxsize=100"
  - "audit-policy-file=/etc/rancher/k3s/audit-policy.yaml"
```

Create the audit policy:

```yaml
# /etc/rancher/k3s/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Log all requests at RequestResponse level
  - level: RequestResponse
    verbs: ["create", "update", "patch", "delete"]
  # Log metadata for read requests
  - level: Metadata
    verbs: ["get", "list", "watch"]
```

## Step 6: Verify FIPS Compliance

```bash
# Verify the host is in FIPS mode
cat /proc/sys/crypto/fips_enabled

# Check which cipher suites are being used by kube-apiserver
# Connect with TLS debugging
openssl s_client -connect <server-ip>:6443 -tls1_2 2>&1 | grep Cipher

# Verify cipher suites in use
nmap --script ssl-enum-ciphers -p 6443 <server-ip>

# Check TLS configuration
kubectl get --raw /apis | head -5

# Verify encryption is active for secrets
kubectl create secret generic test-fips \
  --from-literal=key=value -n default

# Check etcd to confirm secret is encrypted
ETCDCTL_API=3 etcdctl \
  --endpoints https://127.0.0.1:2379 \
  --cacert /var/lib/rancher/k3s/server/tls/etcd/server-ca.crt \
  --cert /var/lib/rancher/k3s/server/tls/etcd/client.crt \
  --key /var/lib/rancher/k3s/server/tls/etcd/client.key \
  get /registry/secrets/default/test-fips | xxd | head
# The output should show encrypted data, not plaintext YAML
```

## Step 7: Container Image FIPS Compliance

Workloads running on FIPS clusters should also use FIPS-compliant base images:

```dockerfile
# Use a FIPS-compliant base image
FROM registry.access.redhat.com/ubi8/ubi-minimal:latest

# UBI (Universal Base Image) from Red Hat supports FIPS mode
# Applications using OpenSSL will use the FIPS-validated module
```

## Step 8: Network Policy for FIPS Environments

```yaml
# Restrict traffic to only FIPS-approved communication
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: fips-default-deny
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  egress:
    # Only allow HTTPS (TLS) egress
    - ports:
        - port: 443
          protocol: TCP
        - port: 6443
          protocol: TCP
```

## Conclusion

Configuring K3s for FIPS 140-2 compliance requires OS-level FIPS enablement, FIPS-validated binaries, and explicit configuration of approved cipher suites and TLS versions. For production FIPS deployments, use Rancher Government Solutions' certified K3s builds and ensure all workloads running on the cluster also use FIPS-validated cryptographic libraries. Always engage your security team and compliance officers to validate the complete configuration against your specific regulatory requirements.
