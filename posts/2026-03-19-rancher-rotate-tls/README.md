# How to Rotate Rancher TLS Certificates

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Security, TLS, Certificate

Description: Learn how to rotate TLS certificates for the Rancher management server to maintain security and avoid certificate expiration.

TLS certificates used by the Rancher management server have expiration dates and should be rotated regularly. Whether you use Let's Encrypt, a custom CA, or self-signed certificates, this guide covers the process for rotating Rancher's TLS certificates before they expire.

## Prerequisites

- Rancher v2.5 or later
- kubectl and Helm 3 access
- Admin access to the Rancher management cluster
- New TLS certificate and key (for custom certificates)

## Step 1: Check Current Certificate Expiration

Before rotating, check when your current certificate expires:

```bash
kubectl get secret tls-rancher-ingress -n cattle-system -o jsonpath='{.data.tls\.crt}' | \
  base64 -d | openssl x509 -noout -dates
```

Output example:

```plaintext
notBefore=Jan 1 00:00:00 2025 GMT
notAfter=Apr 1 00:00:00 2026 GMT
```

You can also check via the Rancher UI by inspecting the browser's certificate details.

## Step 2: Rotate Let's Encrypt Certificates

If you use Let's Encrypt with cert-manager, certificates are rotated automatically. However, you can force a renewal:

```bash
kubectl delete secret tls-rancher-ingress -n cattle-system
```

cert-manager will automatically detect the missing secret and request a new certificate. Monitor the renewal:

```bash
kubectl get certificates -n cattle-system
kubectl describe certificate tls-rancher-ingress -n cattle-system
```

Check cert-manager logs if the renewal does not happen:

```bash
kubectl logs -n cert-manager -l app=cert-manager
```

## Step 3: Rotate Custom TLS Certificates

If you use certificates from your own CA, follow these steps:

### Generate or Obtain a New Certificate

Ensure the new certificate includes the Rancher hostname as a SAN (Subject Alternative Name):

```bash
openssl req -new -newkey rsa:4096 -nodes \
  -keyout rancher-new.key \
  -out rancher-new.csr \
  -subj "/CN=rancher.yourdomain.com" \
  -addext "subjectAltName=DNS:rancher.yourdomain.com"
```

Sign the CSR with your CA or submit it to your certificate authority.

### Update the TLS Secret

Delete the old secret and create a new one:

```bash
kubectl delete secret tls-rancher-ingress -n cattle-system

kubectl create secret tls tls-rancher-ingress \
  -n cattle-system \
  --cert=rancher-new.crt \
  --key=rancher-new.key
```

If you have a CA bundle, update the CA secret as well:

```bash
kubectl delete secret tls-ca -n cattle-system

kubectl create secret generic tls-ca \
  -n cattle-system \
  --from-file=cacerts.pem=ca-chain.crt
```

### Restart Rancher to Pick Up the New Certificate

```bash
kubectl rollout restart deployment rancher -n cattle-system
kubectl rollout status deployment rancher -n cattle-system
```

## Step 4: Update the CA Certificate for Internal Communication

If you changed the CA, update the `cacerts` setting in Rancher:

```bash
kubectl -n cattle-system patch setting cacerts \
  --type='merge' \
  -p "{\"value\":\"$(cat ca-chain.crt)\"}"
```

Or through the Rancher UI:

1. Go to **Global Settings**.
2. Find the **cacerts** setting.
3. Update with the new CA certificate chain.
4. Save.

## Step 5: Update Downstream Cluster Agents

After rotating the Rancher TLS certificate, downstream cluster agents need to trust the new certificate. If the CA changed, update the agents:

```bash
kubectl rollout restart deployment cattle-cluster-agent -n cattle-system
kubectl rollout restart daemonset cattle-node-agent -n cattle-system
```

On each downstream cluster, verify connectivity:

```bash
kubectl logs -n cattle-system -l app=cattle-cluster-agent --tail=20
```

## Step 6: Verify the New Certificate

Check the new certificate is being served:

```bash
echo | openssl s_client -connect rancher.yourdomain.com:443 -servername rancher.yourdomain.com 2>/dev/null | \
  openssl x509 -noout -dates -subject
```

Verify from within the cluster:

```bash
kubectl run cert-check --rm -it --image=alpine --restart=Never -- \
  sh -c 'apk add openssl && echo | openssl s_client -connect rancher.cattle-system.svc:443 2>/dev/null | openssl x509 -noout -dates'
```

## Step 7: Set Up Certificate Expiration Monitoring

Prevent future certificate emergencies by monitoring expiration:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: tls-cert-alerts
  namespace: cattle-monitoring-system
spec:
  groups:
  - name: certificates
    rules:
    - alert: RancherCertExpiringSoon
      expr: |
        (probe_ssl_earliest_cert_expiry - time()) / 86400 < 30
      for: 1h
      labels:
        severity: warning
      annotations:
        summary: "Rancher TLS certificate expires in less than 30 days"
    - alert: RancherCertExpiryCritical
      expr: |
        (probe_ssl_earliest_cert_expiry - time()) / 86400 < 7
      for: 1h
      labels:
        severity: critical
      annotations:
        summary: "Rancher TLS certificate expires in less than 7 days"
```

## Step 8: Automate Certificate Rotation

For ongoing automated rotation, use cert-manager with your own CA:

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: ca-issuer
  namespace: cattle-system
spec:
  ca:
    secretName: ca-key-pair
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: tls-rancher-ingress
  namespace: cattle-system
spec:
  secretName: tls-rancher-ingress
  duration: 2160h  # 90 days
  renewBefore: 360h  # 15 days before expiry
  issuerRef:
    name: ca-issuer
    kind: Issuer
  dnsNames:
  - rancher.yourdomain.com
```

cert-manager will automatically renew the certificate 15 days before expiry.

## Troubleshooting

### Rancher UI Shows Certificate Error After Rotation

Clear your browser cache and cookies. The old certificate may be cached.

### Downstream Clusters Disconnect After CA Change

If the CA changed, downstream agents need the new CA. Re-import the cluster or restart the agents with the new CA.

### cert-manager Fails to Renew

Check cert-manager logs and the Certificate resource status:

```bash
kubectl describe certificate tls-rancher-ingress -n cattle-system
kubectl logs -n cert-manager -l app=cert-manager --tail=50
```

Common issues include DNS validation failures or rate limiting from Let's Encrypt.

## Conclusion

Regular TLS certificate rotation for Rancher prevents service disruptions from expired certificates and maintains trust in your management infrastructure. Whether you use Let's Encrypt for automatic renewal, cert-manager with a custom CA, or manual rotation, establishing a consistent process and monitoring certificate expiration ensures uninterrupted secure access to your Rancher server.
