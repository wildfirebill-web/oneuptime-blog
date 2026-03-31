# How to Implement Data Encryption in Transit with Dapr mTLS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, mTLS, Encryption, Security, Transport Layer Security

Description: Enable and configure Dapr mTLS to encrypt all service-to-service communication in transit, manage certificates, and enforce mutual authentication.

---

Dapr's mTLS (mutual TLS) encrypts all sidecar-to-sidecar communication automatically. When enabled, every service invocation, pub/sub message, and state store call flows through encrypted channels with mutual authentication - the caller and receiver both verify each other's identity. This guide covers enabling, configuring, and verifying mTLS encryption in Dapr.

## How Dapr mTLS Works

Dapr's Sentry component acts as the certificate authority (CA). When a sidecar starts:

1. It generates a private key
2. It requests a certificate from Sentry
3. Sentry issues a short-lived certificate (default 24 hours)
4. All sidecar communication uses these certificates for mTLS

The trust chain:
```text
Sentry (Root CA) -> Sidecar Certificate -> mTLS Connection
```

## Enabling mTLS in Dapr

mTLS is enabled by default in Kubernetes. Verify it is active:

```bash
dapr mtls -k
# Output: Mutual TLS is enabled in your Kubernetes cluster
```

Check certificate status:

```bash
dapr mtls expiry -k
# Output: Certificate expires on: 2026-04-01 16:00:00 +0000 UTC
```

For self-hosted mode, enable mTLS in the Dapr Configuration:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: daprconfig
  namespace: default
spec:
  mtls:
    enabled: true
    workloadCertTTL: 24h
    allowedClockSkew: 15m
```

## Configuring mTLS Certificate Rotation

Dapr automatically rotates certificates before expiry. Configure rotation settings:

```bash
# Check current trust bundle
kubectl get secret dapr-trust-bundle -n dapr-system -o yaml

# Rotate the root certificate manually if needed
dapr mtls renew-certificate -k \
  --ca-root-certificate /path/to/new/ca.crt \
  --issuer-private-key /path/to/new/issuer.key \
  --issuer-public-certificate /path/to/new/issuer.crt
```

For production, use an external CA like Hashicorp Vault:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: daprconfig
spec:
  mtls:
    enabled: true
    rootCA: |
      -----BEGIN CERTIFICATE-----
      ...
      -----END CERTIFICATE-----
```

## Verifying mTLS Encryption

Confirm traffic is encrypted by examining sidecar communication:

```bash
# Check if plain HTTP between sidecars is rejected
kubectl exec -it my-pod -c daprd -- \
  curl -v http://other-service-dapr-sidecar:3500/v1.0/invoke/other-service/method/health

# Successful mTLS connection shows certificate exchange in verbose output
kubectl exec -it my-pod -c daprd -- \
  curl -v --cacert /var/run/secrets/dapr.io/tls/ca.crt \
  https://dapr-sentry.dapr-system.svc.cluster.local:443/healthz
```

Check Sentry logs to confirm certificate issuance:

```bash
kubectl logs -n dapr-system -l app=dapr-sentry --tail=50 | grep "certificate issued"
```

## Enforcing Service Identity with Access Control

mTLS provides identity - combine it with access control:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: order-service-config
spec:
  mtls:
    enabled: true
  accessControl:
    defaultAction: deny
    trustDomain: cluster.local
    policies:
      - appId: payment-service
        defaultAction: allow
        trustDomain: cluster.local
        namespace: default
      - appId: inventory-service
        defaultAction: allow
        trustDomain: cluster.local
        namespace: default
```

## Monitoring mTLS Health

Track certificate expiry and connection errors:

```bash
# Alert if certificate expires within 48 hours
kubectl create -f - <<EOF
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: dapr-cert-expiry
  namespace: monitoring
spec:
  groups:
    - name: dapr.mtls
      rules:
        - alert: DaprCertExpiringSoon
          expr: dapr_sentry_cert_sign_request_received_total > 0
          annotations:
            summary: "Check Dapr certificate expiry"
EOF
```

## Summary

Dapr mTLS encrypts all inter-service communication automatically through the Sentry certificate authority. Configuring short certificate TTLs, enabling external CA integration for production, and combining mTLS with access control policies creates a zero-trust network where every service proves its identity before communicating.
