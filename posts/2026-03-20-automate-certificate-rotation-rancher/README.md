# How to Automate Certificate Rotation in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Certificate, Cert-Manager, TLS, Automation, Kubernetes, Security

Description: Automate TLS certificate rotation in Rancher using cert-manager for application certificates, and built-in RKE2 mechanisms for Kubernetes component certificates, ensuring no certificate expiry...

## Introduction

Certificate expiry is one of the most preventable causes of production outages. Automating certificate rotation-for Kubernetes component certificates (API server, etcd, kubelet), Rancher's own certificates, and application TLS certificates-eliminates manual renewal processes. cert-manager handles application certificates, while RKE2 manages Kubernetes component certificates automatically.

## Step 1: Install cert-manager

```bash
# Install cert-manager with CRDs

helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.14.0 \
  --set installCRDs=true \
  --set prometheus.enabled=true \
  --set webhook.timeoutSeconds=30
```

## Step 2: Configure Certificate Issuers

```yaml
# Let's Encrypt production issuer (for internet-facing services)
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: platform-team@company.com
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
      - http01:
          ingress:
            class: nginx
---
# Internal CA issuer (for internal services)
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: internal-ca
spec:
  ca:
    secretName: internal-ca-key-pair
```

## Step 3: Issue Application Certificates

```yaml
# Certificate for application - auto-renewed 30 days before expiry
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: myapp-tls
  namespace: production
spec:
  secretName: myapp-tls-secret
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  commonName: app.company.com
  dnsNames:
    - app.company.com
    - www.app.company.com
  duration: 2160h        # 90 days
  renewBefore: 720h      # Renew 30 days before expiry
  privateKey:
    rotationPolicy: Always    # Generate new key on each renewal
```

```yaml
# Ingress with automatic cert-manager certificate
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  namespace: production
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - secretName: myapp-tls-secret
      hosts:
        - app.company.com
  rules:
    - host: app.company.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  number: 80
```

## Step 4: Rotate Rancher's TLS Certificate

```bash
# Rancher's own certificate is managed by cert-manager annotation
# To rotate: update the Certificate resource or delete the secret to trigger re-issuance

# Check Rancher certificate expiry
kubectl get certificate -n cattle-system rancher
kubectl get secret tls-rancher-ingress -n cattle-system \
  -o jsonpath='{.data.tls\.crt}' | \
  base64 -d | openssl x509 -noout -dates

# Force renewal before expiry
kubectl annotate certificate rancher \
  cert-manager.io/issue-temporary-certificate="true" \
  -n cattle-system

# Or delete the secret to force re-issuance
kubectl delete secret tls-rancher-ingress -n cattle-system
# cert-manager will automatically re-create it
```

## Step 5: Rotate Kubernetes Component Certificates

RKE2 rotates Kubernetes component certificates automatically and supports manual rotation:

```bash
# Check certificate expiry dates
openssl x509 -noout -dates \
  -in /var/lib/rancher/rke2/server/tls/kube-apiserver/serving-ca.crt

# List all certificates and their expiry
for cert in $(find /var/lib/rancher/rke2/server/tls -name "*.crt" 2>/dev/null); do
  echo "--- $cert ---"
  openssl x509 -noout -subject -dates -in "$cert" 2>/dev/null
done

# Rotate all certificates (requires temporary cluster downtime)
systemctl stop rke2-server

rke2 certificate rotate

systemctl start rke2-server
```

## Step 6: Monitor Certificate Expiry

```yaml
# cert-manager exposes Prometheus metrics
# Alert when certificate expires within 14 days
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: certificate-expiry-alerts
  namespace: cert-manager
spec:
  groups:
    - name: certificates
      rules:
        - alert: CertificateExpiringSoon
          expr: |
            certmanager_certificate_expiration_timestamp_seconds - time() < 1209600
          for: 1h
          annotations:
            summary: "Certificate {{ $labels.name }} in {{ $labels.namespace }} expires in less than 14 days"
          labels:
            severity: warning

        - alert: CertificateExpiryCritical
          expr: |
            certmanager_certificate_expiration_timestamp_seconds - time() < 259200
          for: 1h
          annotations:
            summary: "Certificate {{ $labels.name }} CRITICAL: expires in less than 3 days"
          labels:
            severity: critical

        - alert: CertificateRenewalFailed
          expr: |
            certmanager_certificate_ready_status{condition="False"} == 1
          for: 10m
          annotations:
            summary: "Certificate {{ $labels.name }} renewal is failing"
          labels:
            severity: critical
```

## Step 7: Wildcard Certificate Automation

```yaml
# Wildcard certificate via DNS-01 challenge (doesn't need public HTTP endpoint)
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard-company-com
  namespace: cert-manager
spec:
  secretName: wildcard-company-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - "*.company.com"
    - "*.internal.company.com"
  duration: 2160h
  renewBefore: 720h
```

## Conclusion

cert-manager automates the complete lifecycle of TLS certificates in Rancher-managed clusters. Application certificates are issued automatically when Ingresses are created and renewed before expiry without any manual intervention. RKE2 handles Kubernetes component certificate rotation with a simple `rke2 certificate rotate` command. The combination of cert-manager Prometheus metrics and PrometheusRule alerts ensures certificate expiry never causes unexpected outages-the team is notified weeks before expiry, with critical alerts for any renewal failures.
