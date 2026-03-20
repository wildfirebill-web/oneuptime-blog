# How to Enable Istio mTLS in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Istio, mTLS, Security, Service Mesh

Description: Learn how to enable and configure mutual TLS (mTLS) in Istio to encrypt and authenticate all service-to-service communication in Rancher-managed Kubernetes clusters.

Mutual TLS (mTLS) is one of Istio's most important security features. It automatically encrypts all traffic between services in the mesh and provides strong identity-based authentication using X.509 certificates managed by Istio's certificate authority (istiod). This guide covers how to enable and configure mTLS in a Rancher-managed environment.

## Prerequisites

- Istio installed and running in your Rancher cluster
- Services deployed with Istio sidecar injection enabled
- `kubectl` and `istioctl` access to the cluster

## Understanding Istio mTLS Modes

Istio supports three peer authentication modes:

- **PERMISSIVE**: Accepts both plaintext and mTLS traffic (default, useful for migration)
- **STRICT**: Only accepts mTLS traffic (recommended for production)
- **DISABLE**: Disables mTLS, only accepts plaintext

## Step 1: Check Current mTLS Status

```bash
# Check the current peer authentication policies

kubectl get peerauthentication -A

# Verify the default mesh-wide policy
kubectl get peerauthentication -n istio-system

# Check mTLS status for a specific service using istioctl
istioctl authn tls-check <pod-name>.<namespace>

# Example
istioctl authn tls-check reviews-v1-xxxxxxx.bookinfo
```

## Step 2: Enable Strict mTLS Mesh-Wide

Enable strict mTLS for the entire service mesh:

```yaml
# mesh-wide-mtls.yaml - Enable strict mTLS for all services in the mesh
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  # When applied to istio-system, this becomes the mesh-wide policy
  namespace: istio-system
spec:
  mtls:
    # STRICT: Only accept mTLS connections
    mode: STRICT
```

```bash
# Apply the mesh-wide mTLS policy
kubectl apply -f mesh-wide-mtls.yaml

# Verify the policy was applied
kubectl get peerauthentication -n istio-system
```

## Step 3: Enable mTLS at Namespace Level

Apply mTLS to a specific namespace:

```yaml
# namespace-mtls.yaml - Enable strict mTLS for a specific namespace
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  # Apply to the my-app namespace
  namespace: my-app
spec:
  mtls:
    mode: STRICT
```

```bash
kubectl apply -f namespace-mtls.yaml
```

## Step 4: Enable mTLS at Service Level

For fine-grained control, apply mTLS policies to specific services:

```yaml
# service-mtls.yaml - Configure mTLS for a specific service
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: reviews-mtls
  namespace: bookinfo
spec:
  selector:
    matchLabels:
      # Apply to pods with this label
      app: reviews
  mtls:
    mode: STRICT
  # Override mTLS mode for specific ports
  portLevelMtls:
    "9080":
      mode: STRICT
    "9090":
      # Allow plaintext on port 9090 (e.g., for health checks)
      mode: PERMISSIVE
```

## Step 5: Configure DestinationRule for mTLS

After setting PeerAuthentication, configure the DestinationRule to tell clients to use mTLS:

```yaml
# destination-rule-mtls.yaml - Configure clients to send mTLS traffic
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews-mtls-dr
  namespace: bookinfo
spec:
  host: reviews
  trafficPolicy:
    tls:
      # Use Istio-managed certificates for mTLS
      mode: ISTIO_MUTUAL
```

## Step 6: Gradual Migration from Permissive to Strict mTLS

For production environments, migrate gradually:

```yaml
# Step 1: Ensure namespace is in PERMISSIVE mode (default)
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: my-app
spec:
  mtls:
    mode: PERMISSIVE
```

```bash
# Step 2: Verify all services support mTLS before switching to STRICT
istioctl authn tls-check -n my-app

# Step 3: Look for any services that are still using plaintext
# They will show CLIENT POLICY: NONE or SERVER POLICY: DISABLE
```

```yaml
# Step 4: Switch to STRICT mode after verification
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: my-app
spec:
  mtls:
    mode: STRICT
```

## Step 7: Verify mTLS is Working

```bash
# Check that mTLS is being used between services
istioctl authn tls-check productpage-v1-xxxxxxx.bookinfo reviews.bookinfo.svc.cluster.local

# Expected output showing mTLS is active:
# HOST:PORT              STATUS  SERVER  CLIENT  AUTHN POLICY  DESTINATION RULE
# reviews:9080           OK      mTLS    mTLS    default/bookinfo  reviews/bookinfo

# Use Kiali to visualize mTLS status (look for lock icons on service edges)

# Check the Envoy proxy configuration for TLS settings
istioctl proxy-config cluster deploy/productpage-v1 \
  -n bookinfo | grep reviews

# Verify certificates are being used
kubectl exec -n bookinfo -c istio-proxy \
  $(kubectl get pod -n bookinfo -l app=productpage -o jsonpath='{.items[0].metadata.name}') \
  -- openssl s_client -connect reviews:9080 2>/dev/null | head -20
```

## Step 8: Certificate Management

Istio manages certificates automatically, but you can customize the configuration:

```bash
# View the certificates being used by a service
kubectl exec -n bookinfo -c istio-proxy \
  $(kubectl get pod -n bookinfo -l app=reviews -o jsonpath='{.items[0].metadata.name}') \
  -- openssl x509 -in /var/run/secrets/istio/root-cert.pem -noout -text

# Check certificate expiration
istioctl proxy-config secret deploy/reviews-v1 -n bookinfo
```

## Conclusion

Enabling mTLS in Istio provides a zero-trust security model where all service-to-service communication is encrypted and authenticated without any changes to your application code. By gradually migrating from PERMISSIVE to STRICT mode, you can safely enable mTLS in production environments. Rancher's cluster management capabilities make it easy to monitor and manage these security policies across your entire fleet of Kubernetes clusters.
