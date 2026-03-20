# How to Install Rancher with Let's Encrypt SSL Certificates

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, SSL, Let's Encrypt, Kubernetes, Helm, Installation

Description: A step-by-step guide to installing Rancher with automatic Let's Encrypt SSL certificates using Helm and cert-manager for production-grade HTTPS.

Running Rancher with a valid SSL certificate from Let's Encrypt eliminates browser security warnings and provides trusted HTTPS for your Kubernetes management platform. Let's Encrypt certificates are free, automatically renewable, and widely trusted. This guide covers deploying Rancher on a Kubernetes cluster with automatic Let's Encrypt certificate provisioning using cert-manager.

## Prerequisites

Before you begin, ensure you have:

- A running Kubernetes cluster (v1.25 or later) with at least 3 nodes
- `kubectl` and Helm 3 installed
- A fully qualified domain name (FQDN) with a DNS A record pointing to your cluster's load balancer or ingress IP
- An NGINX Ingress Controller installed on the cluster
- Port 80 open and accessible from the internet (required for Let's Encrypt HTTP-01 challenge)
- A valid email address for Let's Encrypt notifications

## Step 1: Verify DNS Configuration

Before starting, make sure your domain resolves to your cluster's ingress IP:

```bash
# Get your ingress controller's external IP

kubectl get svc -n ingress-nginx

# Verify DNS resolution
nslookup rancher.yourdomain.com
dig rancher.yourdomain.com
```

The DNS record must be active and pointing to the correct IP before Let's Encrypt can issue a certificate.

## Step 2: Install cert-manager

cert-manager handles the automatic issuance and renewal of Let's Encrypt certificates.

Apply the cert-manager CRDs:

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.4/cert-manager.crds.yaml
```

Add the Jetstack Helm repository:

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
```

Install cert-manager:

```bash
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.14.4
```

Wait for cert-manager to be ready:

```bash
kubectl get pods -n cert-manager
```

All pods should be in the Running state before proceeding.

## Step 3: Create a Let's Encrypt ClusterIssuer

Create a ClusterIssuer resource that tells cert-manager how to request certificates from Let's Encrypt. Start with the staging issuer for testing:

```yaml
# letsencrypt-staging.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
      - http01:
          ingress:
            class: nginx
```

Apply it:

```bash
kubectl apply -f letsencrypt-staging.yaml
```

Once you have verified that everything works with the staging issuer, create the production issuer:

```yaml
# letsencrypt-production.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-production
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-production
    solvers:
      - http01:
          ingress:
            class: nginx
```

Apply it:

```bash
kubectl apply -f letsencrypt-production.yaml
```

Verify the issuer is ready:

```bash
kubectl get clusterissuer
```

Both issuers should show `Ready: True`.

## Step 4: Add the Rancher Helm Repository

```bash
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update
```

## Step 5: Create the cattle-system Namespace

```bash
kubectl create namespace cattle-system
```

## Step 6: Install Rancher with Let's Encrypt

First test with the staging issuer to avoid hitting Let's Encrypt rate limits:

```bash
helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --set hostname=rancher.yourdomain.com \
  --set bootstrapPassword=yourSecurePassword \
  --set replicas=3 \
  --set ingress.tls.source=letsEncrypt \
  --set letsEncrypt.email=your-email@example.com \
  --set letsEncrypt.ingress.class=nginx \
  --set letsEncrypt.environment=staging
```

Wait for the deployment:

```bash
kubectl rollout status deployment rancher -n cattle-system
```

Check the certificate status:

```bash
kubectl get certificate -n cattle-system
kubectl describe certificate tls-rancher-ingress -n cattle-system
```

## Step 7: Switch to Production Certificates

Once the staging certificate is successfully issued, upgrade to production:

```bash
# Delete the staging certificate and secret
kubectl delete certificate tls-rancher-ingress -n cattle-system
kubectl delete secret tls-rancher-ingress -n cattle-system

# Upgrade Rancher to use production Let's Encrypt
helm upgrade rancher rancher-stable/rancher \
  --namespace cattle-system \
  --set hostname=rancher.yourdomain.com \
  --set bootstrapPassword=yourSecurePassword \
  --set replicas=3 \
  --set ingress.tls.source=letsEncrypt \
  --set letsEncrypt.email=your-email@example.com \
  --set letsEncrypt.ingress.class=nginx \
  --set letsEncrypt.environment=production
```

## Step 8: Verify the Certificate

Check that the production certificate has been issued:

```bash
kubectl get certificate -n cattle-system
kubectl describe certificate tls-rancher-ingress -n cattle-system
```

The certificate should show `Ready: True` and the issuer should be `letsencrypt-production`.

You can also verify from the command line:

```bash
echo | openssl s_client -connect rancher.yourdomain.com:443 -servername rancher.yourdomain.com 2>/dev/null | openssl x509 -noout -issuer -dates
```

## Step 9: Access the Rancher UI

Navigate to `https://rancher.yourdomain.com` in your browser. You should see a valid SSL certificate with no browser warnings. Log in with the bootstrap password and complete the initial setup.

## Using DNS-01 Challenge

If port 80 is not available or you need wildcard certificates, use the DNS-01 challenge instead. This example uses Cloudflare as the DNS provider:

```yaml
# letsencrypt-dns.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-dns
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-dns
    solvers:
      - dns01:
          cloudflare:
            email: your-cloudflare-email@example.com
            apiTokenSecretRef:
              name: cloudflare-api-token
              key: api-token
```

Create the Cloudflare API token secret:

```bash
kubectl create secret generic cloudflare-api-token \
  --namespace cert-manager \
  --from-literal=api-token=YOUR_CLOUDFLARE_API_TOKEN
```

## Certificate Renewal

Let's Encrypt certificates are valid for 90 days. cert-manager automatically renews them 30 days before expiration. Monitor the renewal status:

```bash
kubectl get certificate -n cattle-system
kubectl describe certificate tls-rancher-ingress -n cattle-system | grep -A5 "Renewal"
```

## Troubleshooting

```bash
# Check cert-manager logs
kubectl logs -l app=cert-manager -n cert-manager --tail=50

# Check certificate status
kubectl describe certificate tls-rancher-ingress -n cattle-system

# Check certificate request
kubectl get certificaterequest -n cattle-system
kubectl describe certificaterequest -n cattle-system

# Check ACME challenges
kubectl get challenges -n cattle-system

# Verify ingress configuration
kubectl get ingress -n cattle-system -o yaml

# Test HTTP-01 challenge accessibility
curl -v http://rancher.yourdomain.com/.well-known/acme-challenge/test
```

Common issues:

- **DNS not resolving**: Ensure the A record is properly configured
- **Port 80 blocked**: The HTTP-01 challenge requires port 80 to be open
- **Rate limits**: Let's Encrypt has rate limits. Use the staging environment for testing
- **Ingress class mismatch**: Verify the ingress class matches your ingress controller

## Conclusion

You have successfully installed Rancher with automatic Let's Encrypt SSL certificates. Your Rancher instance now has trusted HTTPS that automatically renews before expiration. This setup is ideal for production environments where browser trust and secure communication are essential. The combination of cert-manager and Let's Encrypt provides a maintenance-free SSL solution for your Kubernetes management platform.
