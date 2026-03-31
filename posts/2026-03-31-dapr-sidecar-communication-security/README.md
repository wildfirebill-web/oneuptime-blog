# How to Secure Dapr Sidecar Communication

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Security, mTLS, Sidecar, Certificate

Description: Learn how Dapr secures sidecar-to-sidecar communication with mTLS, how to manage certificates, and how to customize TLS settings for your environment.

---

## Overview

Dapr secures all communication between sidecars using mutual TLS (mTLS) by default in Kubernetes mode. Each sidecar receives a SPIFFE-compliant X.509 certificate issued by the Dapr control plane, and both sides of every connection verify each other's certificates before exchanging data. This prevents eavesdropping and impersonation attacks.

## How Dapr mTLS Works

The Dapr `sentry` service acts as a Certificate Authority (CA). When a sidecar starts, it generates a key pair, submits a Certificate Signing Request (CSR) to `sentry`, and receives a signed certificate with a short TTL (default: 24 hours). The certificate is automatically rotated before it expires.

The `operator` and `placement` services also use mTLS, so the entire control plane is protected.

## Verifying mTLS Status

Check the system configuration to confirm mTLS is enabled:

```bash
kubectl get configuration daprsystem -n dapr-system -o yaml
```

Expected output includes:

```yaml
spec:
  mtls:
    enabled: true
    workloadCertTTL: "24h"
    allowedClockSkew: "15m"
```

## Disabling mTLS (Development Only)

Never disable mTLS in production. For local development without Kubernetes:

```bash
dapr run --app-id service-a --app-port 3000 \
  --dapr-http-port 3500 -- node app.js
```

In self-hosted mode, mTLS is disabled by default because all communication is on localhost.

To explicitly disable mTLS in Kubernetes (not recommended):

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: daprsystem
  namespace: dapr-system
spec:
  mtls:
    enabled: false
```

## Customizing Certificate TTL

Shorter certificate lifetimes reduce the window of exposure if a certificate is compromised:

```yaml
spec:
  mtls:
    enabled: true
    workloadCertTTL: "1h"
    allowedClockSkew: "5m"
```

Apply via the Dapr control plane configuration:

```bash
helm upgrade dapr dapr/dapr \
  --namespace dapr-system \
  --set dapr_sentry.config.workloadCertTTL=1h
```

## Using Custom Root Certificates

To use your own CA instead of the Dapr-generated one, provide the certificates as a Kubernetes secret before installing Dapr:

```bash
kubectl create secret generic dapr-trust-bundle \
  --from-file=ca.crt=./ca.crt \
  --from-file=issuer.crt=./issuer.crt \
  --from-file=issuer.key=./issuer.key \
  -n dapr-system
```

Then install Dapr with the existing certs flag:

```bash
helm install dapr dapr/dapr \
  --namespace dapr-system \
  --set-string dapr_sentry.config.existingRootCertificate=true
```

## Monitoring Certificate Expiry

Check certificate status using the Dapr CLI:

```bash
dapr mtls check --kubernetes
dapr mtls expiry --kubernetes
```

## Summary

Dapr mTLS provides strong cryptographic identity and encrypted transport for all sidecar-to-sidecar communication with zero application code changes. Managing certificate TTLs and optionally supplying your own CA gives you full control over the trust hierarchy in production environments.
