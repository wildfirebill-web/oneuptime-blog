# How to Configure RKE2 FIPS Compliance Mode

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RKE2, Kubernetes, FIPS, Compliance, Security, Government

Description: Learn how to enable and configure FIPS 140-2 compliance mode in RKE2 for government and regulated industry Kubernetes deployments.

FIPS 140-2 (Federal Information Processing Standard) is a US government standard specifying cryptographic module requirements. It is required for many federal agency workloads and is increasingly required in financial and healthcare regulated environments. RKE2 supports FIPS 140-2 compliant mode, using only approved cryptographic algorithms. This guide covers enabling and verifying FIPS mode in RKE2.

## Prerequisites

- A FIPS-enabled Linux operating system (RHEL 8/9 with FIPS mode, Ubuntu 20.04 with FIPS kernel)
- RKE2 v1.21+ (FIPS builds are available)
- Understanding of FIPS cryptographic requirements

## Understanding FIPS in RKE2

When FIPS mode is enabled in RKE2:

- All cryptographic operations use FIPS 140-2 approved algorithms
- TLS connections use only FIPS-approved cipher suites
- Key generation uses FIPS-compliant methods
- Binary is compiled with FIPS-validated cryptographic modules (BoringCrypto)

Approved algorithms include:
- **Symmetric encryption**: AES-128, AES-256
- **Hash functions**: SHA-256, SHA-384, SHA-512
- **Key exchange**: ECDH (P-256, P-384), RSA
- **Digital signatures**: ECDSA, RSA (2048-bit minimum)

## Step 1: Enable FIPS on the Operating System

### RHEL 8/9 (Recommended for FIPS)

```bash
# Enable FIPS mode on RHEL 8/9
sudo fips-mode-setup --enable

# Reboot to apply FIPS mode
sudo reboot

# After reboot, verify FIPS is enabled
sudo fips-mode-setup --check
# Expected: FIPS mode is enabled

# Additional verification
cat /proc/sys/crypto/fips_enabled
# Expected output: 1
```

### Ubuntu 20.04/22.04

```bash
# Enable FIPS on Ubuntu (requires Ubuntu Pro)
sudo ua enable fips

# Reboot after enabling
sudo reboot

# Verify FIPS is enabled
cat /proc/sys/crypto/fips_enabled
# Expected: 1

# Check FIPS kernel
uname -r
# Should show fips in kernel name or
# check openssl fips mode
openssl md5 /dev/null 2>&1 | grep -i error || echo "FIPS active - MD5 disabled"
```

## Step 2: Download RKE2 FIPS Build

```bash
# RKE2 FIPS builds are separate from standard builds
# The FIPS build suffix is typically "-fips"
RKE2_VERSION="v1.28.8+rke2r1"

# Download the FIPS-compliant RKE2 installation
curl -sfL https://get.rke2.io | \
  INSTALL_RKE2_VERSION="${RKE2_VERSION}" \
  sh -

# For FIPS-specific builds, check the RKE2 FIPS release page
# or use channel: fips
curl -sfL https://get.rke2.io | \
  INSTALL_RKE2_CHANNEL=stable \
  INSTALL_RKE2_TYPE=server \
  sh -
```

## Step 3: Configure RKE2 for FIPS Mode

```yaml
# /etc/rancher/rke2/config.yaml - FIPS configuration
# Note: FIPS mode in RKE2 is primarily handled at the binary level
# and through correct TLS configuration

kube-apiserver-arg:
  # FIPS-compliant TLS cipher suites only
  - "tls-min-version=VersionTLS12"
  - "tls-cipher-suites=TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305"

  # Disable non-FIPS auth methods
  - "anonymous-auth=false"

kube-controller-manager-arg:
  - "tls-min-version=VersionTLS12"
  - "tls-cipher-suites=TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384"

kubelet-arg:
  - "tls-min-version=VersionTLS12"
  - "tls-cipher-suites=TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384"

# Use the CIS hardened profile (complements FIPS)
profile: cis-1.23
```

## Step 4: Verify FIPS Cryptography in Use

```bash
# Check that the RKE2 binary uses FIPS-approved crypto
# RKE2 FIPS builds use BoringCrypto, which is FIPS 140-2 validated

# Verify the go binary was compiled with FIPS support
file /usr/local/bin/rke2 | grep -i "not stripped" && \
  strings /usr/local/bin/rke2 | grep -i "fips" | head -5

# Check TLS configuration on API server
openssl s_client -connect localhost:6443 -status 2>&1 | grep -E "TLS|Cipher"

# Verify only FIPS cipher suites are available
openssl s_client -connect localhost:6443 2>&1 | grep "Cipher is"

# Test that non-FIPS algorithms are rejected
openssl s_client -connect localhost:6443 \
  -cipher RC4-MD5 2>&1 | grep -E "handshake failure|no cipher"
# Should show handshake failure (non-FIPS cipher rejected)
```

## Step 5: Configure Applications for FIPS Compliance

Applications running on a FIPS-enabled RKE2 cluster should also use FIPS-compliant cryptography:

```yaml
# fips-compliant-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fips-app
  namespace: secure-workloads
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fips-app
  template:
    metadata:
      labels:
        app: fips-app
    spec:
      # Use a FIPS-compiled application image
      containers:
      - name: fips-app
        # Use a RHEL UBI base image with FIPS mode
        image: registry.access.redhat.com/ubi8/ubi:latest
        env:
        # Enable FIPS mode in the container
        - name: OPENSSL_FIPS
          value: "1"
        securityContext:
          runAsNonRoot: true
          allowPrivilegeEscalation: false
          seccompProfile:
            type: RuntimeDefault
          capabilities:
            drop: ["ALL"]
```

## Step 6: FIPS Documentation and Attestation

```bash
# Generate FIPS compliance report
cat > /tmp/fips-compliance-check.sh << 'EOF'
#!/bin/bash
echo "=== FIPS Compliance Check for RKE2 ==="
echo "Date: $(date)"
echo ""

echo "1. OS FIPS Mode:"
cat /proc/sys/crypto/fips_enabled
fips-mode-setup --check 2>/dev/null || echo "fips-mode-setup not available"

echo ""
echo "2. Kernel FIPS Support:"
uname -r
ls /boot/vmlinuz-$(uname -r) 2>/dev/null | xargs file 2>/dev/null

echo ""
echo "3. OpenSSL FIPS Status:"
openssl md5 /dev/null 2>&1 && echo "WARNING: MD5 available (not FIPS)" || echo "MD5 disabled (FIPS mode)"

echo ""
echo "4. RKE2 TLS Configuration:"
openssl s_client -connect 127.0.0.1:6443 2>&1 | grep "Cipher\|Protocol" | head -5

echo ""
echo "5. System Crypto Policies:"
update-crypto-policies --show 2>/dev/null || echo "update-crypto-policies not available"
EOF

chmod +x /tmp/fips-compliance-check.sh
sudo /tmp/fips-compliance-check.sh
```

## Conclusion

Enabling FIPS mode in RKE2 involves both OS-level FIPS configuration and RKE2-specific TLS cipher suite restrictions. The combination ensures all cryptographic operations in the Kubernetes cluster use only FIPS 140-2 approved algorithms. For government customers and regulated industries, starting with a FIPS-enabled OS (RHEL with FIPS mode or Ubuntu with FIPS kernel) and the RKE2 FIPS build provides the foundation for a fully compliant Kubernetes environment. Remember that FIPS compliance is a system-wide requirement — applications deployed on the cluster must also use FIPS-compliant cryptography.
