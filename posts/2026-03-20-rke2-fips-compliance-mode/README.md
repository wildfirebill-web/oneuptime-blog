# How to Configure RKE2 FIPS Compliance Mode - Mode

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RKE2, FIPS, Compliance, Security, Kubernetes, Government, SUSE Rancher

Description: Learn how to enable FIPS 140-2 compliant mode in RKE2 to meet federal government security requirements, including OS preparation, RKE2 FIPS binary installation, and verification.

---

FIPS 140-2 compliance is required for federal government deployments and many regulated industries. RKE2 provides a dedicated FIPS-enabled build that uses only FIPS-approved cryptographic modules.

---

## Step 1: Enable FIPS Mode on the Host OS

FIPS mode must be enabled at the OS level before installing RKE2:

```bash
# Enable FIPS on RHEL/CentOS/Rocky Linux

sudo fips-mode-setup --enable
sudo reboot

# Verify FIPS is active after reboot
fips-mode-setup --check
# Expected output: FIPS mode is enabled
```

For Ubuntu:

```bash
# Install FIPS packages (requires Ubuntu Advantage or Pro subscription)
sudo ua enable fips
sudo reboot
```

---

## Step 2: Install the RKE2 FIPS Binary

RKE2 provides a dedicated FIPS build. Install it by setting the channel to `stable` and using the FIPS installer flag:

```bash
# Install RKE2 with FIPS-validated crypto
curl -sfL https://get.rke2.io | \
  INSTALL_RKE2_CHANNEL=stable \
  INSTALL_RKE2_TYPE=server \
  sh -

# Alternatively, download the FIPS binary directly
curl -Lo /usr/local/bin/rke2 \
  https://github.com/rancher/rke2/releases/download/v1.30.2+rke2r1/rke2.linux-amd64-fips
chmod +x /usr/local/bin/rke2
```

---

## Step 3: Configure RKE2 for FIPS

```yaml
# /etc/rancher/rke2/config.yaml
token: my-fips-cluster-token
tls-san:
  - "rke2-fips.example.com"

# Disable non-FIPS algorithms
kube-apiserver-arg:
  - "tls-min-version=VersionTLS12"
  - "tls-cipher-suites=TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384"

kube-controller-manager-arg:
  - "tls-min-version=VersionTLS12"
  - "tls-cipher-suites=TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256"

kubelet-arg:
  - "tls-min-version=VersionTLS12"
  - "tls-cipher-suites=TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256"
```

---

## Step 4: Start RKE2 and Verify

```bash
systemctl enable --now rke2-server.service

# Verify the FIPS binary is in use
/usr/local/bin/rke2 --version | grep fips

# Check TLS negotiation uses only FIPS-approved ciphers
openssl s_client -connect 127.0.0.1:6443 -tls1_2 2>&1 | grep Cipher
```

---

## Step 5: Run CIS Benchmark Alongside FIPS

FIPS-enabled clusters should also meet the CIS Kubernetes Benchmark:

```yaml
# Add to config.yaml
profile: cis-1.23
```

---

## Verification Checklist

- [ ] Host OS FIPS mode enabled and verified after reboot
- [ ] RKE2 FIPS binary installed (not the standard binary)
- [ ] TLS minimum version set to 1.2
- [ ] Only FIPS-approved cipher suites configured
- [ ] etcd encrypted with AES-256-GCM
- [ ] All container images use FIPS-validated Go builds

---

## Best Practices

- Purchase SUSE's FIPS-validated Rancher Prime subscription for formal compliance documentation.
- Test FIPS mode in a non-production environment first - some third-party software may break due to algorithm restrictions.
- Document your crypto module versions and certification numbers for auditors.
