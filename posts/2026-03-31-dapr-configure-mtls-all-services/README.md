# How to Configure Mutual TLS (mTLS) Between All Dapr Services

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, mTLS, Security, Certificate, Encryption, Kubernetes

Description: Learn how to configure mutual TLS between all Dapr services in Kubernetes, including enabling mTLS globally, verifying certificate issuance, and troubleshooting handshake failures.

---

Mutual TLS (mTLS) ensures that both sides of every service-to-service connection authenticate with certificates. Dapr enables mTLS by default in Kubernetes mode through its Sentry certificate authority service. This guide covers full configuration and verification.

## How Dapr mTLS Works

Dapr Sentry is the internal CA that issues short-lived X.509 certificates to each sidecar. When Service A calls Service B, the sidecars perform a mutual TLS handshake - each side verifies the other's certificate before forwarding the request.

## Enable mTLS via Dapr Configuration

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: dapr-config
  namespace: default
spec:
  mtls:
    enabled: true
    workloadCertTTL: "24h"
    allowedClockSkew: "15m"
```

Apply the configuration:

```bash
kubectl apply -f dapr-config.yaml

# Reference this configuration in your deployment
kubectl annotate pod my-pod dapr.io/config=dapr-config
```

## Annotate Deployments to Use the Configuration

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 2
  template:
    metadata:
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "order-service"
        dapr.io/app-port: "8080"
        dapr.io/config: "dapr-config"
    spec:
      containers:
      - name: order-service
        image: myrepo/order-service:latest
```

## Verify mTLS Certificate Issuance

```bash
# Check that Sentry is running
kubectl get pods -n dapr-system -l app=dapr-sentry

# Inspect the trust bundle
kubectl get secret dapr-trust-bundle -n dapr-system -o jsonpath='{.data.issuer\.crt}' | \
  base64 -d | openssl x509 -text -noout

# Verify sidecar received its certificate
kubectl exec deployment/order-service -c daprd -- \
  cat /var/run/secrets/dapr.io/tls/cert.pem | openssl x509 -text -noout | \
  grep -E "Subject|Issuer|Not After"
```

## Check mTLS Status with Dapr CLI

```bash
# View the mTLS configuration
dapr mtls -k

# List expiry dates for all workload certificates
dapr mtls expiry -k

# Export the root certificate
dapr mtls export -k -o ./certs
```

## Troubleshooting mTLS Failures

```bash
# Check Sentry logs for certificate issuance errors
kubectl logs -n dapr-system -l app=dapr-sentry --tail=100

# Check sidecar logs for handshake failures
kubectl logs deployment/order-service -c daprd | grep -i "tls\|certificate\|handshake"

# Verify Sentry is reachable from the sidecar
kubectl exec deployment/order-service -c daprd -- \
  curl -k https://dapr-sentry.dapr-system.svc.cluster.local:443/healthz
```

## Disable mTLS for Specific Namespaces (Testing Only)

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: dapr-config-no-mtls
  namespace: test
spec:
  mtls:
    enabled: false
```

## Summary

Dapr mTLS is managed by the Sentry CA service, which automatically issues short-lived certificates to every sidecar. Enable it via a Configuration resource and reference it in deployment annotations. Use `dapr mtls` CLI commands to inspect certificate status and expiry. Always keep mTLS enabled in production environments.
