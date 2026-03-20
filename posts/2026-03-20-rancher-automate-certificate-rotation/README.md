# How to Automate Certificate Rotation in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: rancher, certificates, tls, automation, cert-manager, security

Description: A guide to automating TLS certificate rotation in Rancher environments using cert-manager, RKE2 certificate rotation, and automated renewal workflows.

## Overview

TLS certificates have expiry dates, and certificate expiration is a common cause of outages in Kubernetes environments. Automating certificate rotation for both the Rancher management server and managed clusters is essential for operational reliability. This guide covers automating certificate rotation using cert-manager, RKE2's built-in rotation commands, and monitoring for certificate expiry.

## Rancher Server Certificate Rotation

### Using cert-manager for Automatic Renewal

The recommended approach for Rancher TLS is to use cert-manager, which automatically renews certificates before they expire.

```bash
# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# Wait for cert-manager to be ready
kubectl wait --for=condition=ready pod \
  -l app=cert-manager \
  -n cert-manager \
  --timeout=120s
```

### Install Rancher with cert-manager

```bash
# Install Rancher using cert-manager for automatic certificate management
helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --set hostname=rancher.example.com \
  --set ingress.tls.source=letsEncrypt \
  --set letsEncrypt.email=admin@example.com \
  --set letsEncrypt.environment=production
```

### ClusterIssuer Configuration

```yaml
# ClusterIssuer for Let's Encrypt production certificates
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
      - http01:
          ingress:
            class: nginx
```

```yaml
# For internal CA (enterprise environments)
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: internal-ca-issuer
spec:
  ca:
    secretName: internal-ca-secret
---
# Certificate resource - cert-manager renews 30 days before expiry
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: rancher-tls
  namespace: cattle-system
spec:
  secretName: tls-rancher-ingress
  issuerRef:
    name: internal-ca-issuer
    kind: ClusterIssuer
  dnsNames:
    - rancher.example.com
  duration: 2160h      # 90 days
  renewBefore: 720h    # Renew 30 days before expiry
```

## RKE2 Cluster Certificate Rotation

RKE2 manages its own cluster certificates and provides a built-in rotation command:

### Check Certificate Expiry

```bash
# Check when cluster certificates expire
# On RKE2 server node:
rke2 certificate rotate --help

# View current certificate expiry dates
for cert in /var/lib/rancher/rke2/server/tls/*.crt; do
  echo "Certificate: $cert"
  openssl x509 -in "$cert" -noout -enddate 2>/dev/null
  echo "---"
done
```

### Manual Certificate Rotation

```bash
# Stop RKE2 before rotation
systemctl stop rke2-server

# Rotate all certificates
rke2 certificate rotate

# Start RKE2 with new certificates
systemctl start rke2-server
```

### Automated Certificate Rotation Script

```bash
#!/bin/bash
# auto-rotate-rke2-certs.sh
# Run as a CronJob 30 days before expiry

CERT_FILE="/var/lib/rancher/rke2/server/tls/server-ca.crt"
DAYS_THRESHOLD=30

# Check expiry
EXPIRY=$(openssl x509 -in "${CERT_FILE}" -noout -enddate | cut -d= -f2)
EXPIRY_EPOCH=$(date -d "${EXPIRY}" +%s)
NOW_EPOCH=$(date +%s)
DAYS_REMAINING=$(( (EXPIRY_EPOCH - NOW_EPOCH) / 86400 ))

echo "Certificate expires in ${DAYS_REMAINING} days"

if [ "${DAYS_REMAINING}" -lt "${DAYS_THRESHOLD}" ]; then
  echo "Certificate expiry within ${DAYS_THRESHOLD} days. Starting rotation..."

  # Notify operators
  curl -X POST "${SLACK_WEBHOOK}" \
    -d "{\"text\": \"RKE2 certificate rotation starting on ${HOSTNAME}\"}"

  # Stop RKE2 (do this on all nodes in order)
  systemctl stop rke2-server

  # Rotate certificates
  rke2 certificate rotate

  # Start RKE2
  systemctl start rke2-server

  # Wait for cluster to be healthy
  sleep 60
  kubectl get nodes

  echo "Certificate rotation complete"
  curl -X POST "${SLACK_WEBHOOK}" \
    -d "{\"text\": \"RKE2 certificate rotation completed on ${HOSTNAME}\"}"
fi
```

### Schedule as a CronJob

```yaml
# CronJob to check and rotate certificates monthly
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cert-rotation-check
  namespace: kube-system
spec:
  schedule: "0 2 1 * *"    # Monthly on the 1st at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          hostPID: true
          serviceAccountName: cert-rotation-sa
          containers:
            - name: cert-checker
              image: registry.example.com/cert-rotation-tool:latest
              env:
                - name: NODE_NAME
                  valueFrom:
                    fieldRef:
                      fieldPath: spec.nodeName
              securityContext:
                privileged: true    # Required for RKE2 operations
          nodeSelector:
            node-role.kubernetes.io/control-plane: "true"
          restartPolicy: OnFailure
          tolerations:
            - key: "node-role.kubernetes.io/control-plane"
              operator: "Exists"
              effect: "NoSchedule"
```

## Monitoring Certificate Expiry

### Prometheus Alert for Certificate Expiry

```yaml
# Alert 30 days before Rancher certificate expires
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: cert-expiry-alerts
  namespace: cattle-monitoring-system
spec:
  groups:
    - name: certificate-expiry
      rules:
        # Alert if any certificate on Rancher ingress expires within 30 days
        - alert: CertificateExpiringSoon
          expr: |
            (ssl_certificate_expiry_seconds{job="rancher-monitoring"} - time()) / 86400 < 30
          for: 1h
          labels:
            severity: warning
          annotations:
            summary: "Certificate expiring in {{ $value | humanizeDuration }}"
            description: "Certificate for {{ $labels.server_name }} expires soon"

        # Critical: expires within 7 days
        - alert: CertificateCriticalExpiry
          expr: |
            (ssl_certificate_expiry_seconds{job="rancher-monitoring"} - time()) / 86400 < 7
          for: 1h
          labels:
            severity: critical
          annotations:
            summary: "Certificate expiring in less than 7 days!"
```

## Blackbox Exporter for Certificate Monitoring

```yaml
# Deploy blackbox-exporter for TLS probe
apiVersion: v1
kind: ConfigMap
metadata:
  name: blackbox-config
  namespace: cattle-monitoring-system
data:
  blackbox.yml: |
    modules:
      https_rancher:
        prober: http
        timeout: 10s
        http:
          valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
          tls_config:
            insecure_skip_verify: false
          preferred_ip_protocol: ip4
```

## Conclusion

Automating certificate rotation in Rancher prevents the certificate expiry outages that are unfortunately common in Kubernetes environments. Using cert-manager for Rancher's own TLS certificates provides hands-free renewal. RKE2's built-in certificate rotation command simplifies cluster certificate management. Combining Prometheus certificate expiry monitoring with alerting ensures you have advance warning before any certificate expires. Test your certificate rotation procedures in a non-production environment before applying them to production.
