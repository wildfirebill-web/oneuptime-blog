# How to Troubleshoot Certificate Errors in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Troubleshooting, Certificates, TLS

Description: A comprehensive guide to diagnosing and resolving TLS certificate errors in Rancher, covering cert-manager, self-signed CAs, and certificate rotation.

## Introduction

Certificate errors are among the most disruptive issues in a Rancher deployment. They can prevent the UI from loading, block agent connections, and cause cascading failures across clusters. This guide covers how to identify, diagnose, and fix the most common certificate problems.

## Common Certificate Error Symptoms

- Browser shows "Your connection is not private" (ERR_CERT_AUTHORITY_INVALID)
- Rancher agents log `x509: certificate signed by unknown authority`
- `kubectl` commands fail with `certificate has expired or is not yet valid`
- Rancher UI shows clusters as "Unavailable"

## Step 1: Inspect the Current Certificate

```bash
# Check certificate details from the command line
echo | openssl s_client -connect <rancher-hostname>:443 2>/dev/null \
  | openssl x509 -noout -text | grep -E "Subject:|Issuer:|Not Before:|Not After:"

# Check the Kubernetes TLS secret directly
kubectl get secret -n cattle-system tls-rancher-ingress -o json \
  | jq -r '.data["tls.crt"]' | base64 -d \
  | openssl x509 -noout -dates -subject -issuer
```

## Step 2: Check cert-manager Status

Most Rancher installations use cert-manager to issue and renew certificates automatically.

```bash
# Check cert-manager pods
kubectl get pods -n cert-manager

# List all Certificate resources
kubectl get certificates -A

# Inspect the Rancher certificate
kubectl describe certificate -n cattle-system tls-rancher-ingress

# Check CertificateRequests for renewal status
kubectl get certificaterequest -n cattle-system
kubectl describe certificaterequest -n cattle-system <request-name>

# Check cert-manager logs for errors
kubectl logs -n cert-manager -l app=cert-manager --tail=100
```

## Step 3: Force Certificate Renewal

```bash
# Annotate the certificate to trigger immediate renewal
kubectl annotate certificate -n cattle-system tls-rancher-ingress \
  cert-manager.io/issue-temporary-certificate=true

# Or delete the secret to force re-issuance (cert-manager will recreate it)
kubectl delete secret -n cattle-system tls-rancher-ingress

# Watch the renewal progress
kubectl get certificate -n cattle-system -w
```

## Step 4: Troubleshoot Let's Encrypt Issuance

```bash
# Check the ClusterIssuer configuration
kubectl get clusterissuer letsencrypt-prod -o yaml

# Verify ACME account registration
kubectl describe clusterissuer letsencrypt-prod | grep -A5 "Status:"

# Check the ACME Challenge
kubectl get challenge -n cattle-system

# The Let's Encrypt DNS-01 or HTTP-01 challenge must succeed
# HTTP-01 requires port 80 to be accessible from the internet
curl -v http://<rancher-hostname>/.well-known/acme-challenge/test
```

## Step 5: Troubleshoot Self-Signed or Private CA

```bash
# If using a private CA, check the CA secret
kubectl get secret -n cattle-system tls-rancher-ingress-ca -o json \
  | jq -r '.data["tls.crt"]' | base64 -d \
  | openssl x509 -noout -subject -issuer -dates

# Verify the CA cert matches the server cert's issuer
ISSUER=$(kubectl get secret -n cattle-system tls-rancher-ingress -o json \
  | jq -r '.data["tls.crt"]' | base64 -d \
  | openssl x509 -noout -issuer)
echo "Server cert issuer: $ISSUER"

# Distribute the private CA to client machines
# macOS:
sudo security add-trusted-cert -d -r trustRoot \
  -k /Library/Keychains/System.keychain ca.pem

# Ubuntu/Debian:
sudo cp ca.pem /usr/local/share/ca-certificates/rancher-ca.crt
sudo update-ca-certificates
```

## Step 6: Rotate Certificates Manually

For situations where you need to replace certificates with new ones:

```bash
# Create a new TLS secret with your updated certificates
kubectl create secret tls tls-rancher-ingress \
  --cert=new-cert.pem \
  --key=new-key.pem \
  -n cattle-system \
  --dry-run=client -o yaml | kubectl apply -f -

# Restart Rancher to pick up the new certificate
kubectl rollout restart deployment/rancher -n cattle-system

# Watch the rollout
kubectl rollout status deployment/rancher -n cattle-system
```

## Step 7: Update the Cattle CA Checksum

When the CA certificate changes, agents must be updated with the new checksum:

```bash
# Calculate the new CA checksum
CA_CHECKSUM=$(kubectl get secret -n cattle-system tls-rancher-ingress -o json \
  | jq -r '.data["tls.crt"]' | base64 -d | sha256sum | awk '{print $1}')

# Update the cattle-cluster-agent with the new checksum
kubectl set env deployment/cattle-cluster-agent -n cattle-system \
  CATTLE_CA_CHECKSUM="${CA_CHECKSUM}"

# Restart the agent
kubectl rollout restart deployment/cattle-cluster-agent -n cattle-system
```

## Conclusion

Certificate errors in Rancher cascade quickly, impacting UI access, agent connectivity, and cluster availability. The key is to quickly determine whether the issue is expiry, trust, or issuance — then target the right solution. Keep cert-manager healthy, monitor certificate expiry proactively (Rancher's monitoring stack can alert on this), and ensure all agents trust your CA to maintain a smooth-running environment.
