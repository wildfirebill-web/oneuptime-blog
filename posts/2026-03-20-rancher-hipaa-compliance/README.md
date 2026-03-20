# How to Implement HIPAA Compliance with Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, HIPAA, Compliance, Healthcare, Security

Description: Learn how to configure Rancher-managed Kubernetes clusters to meet HIPAA technical safeguard requirements for protecting electronic Protected Health Information (ePHI).

HIPAA (Health Insurance Portability and Accountability Act) requires healthcare organizations to implement specific technical safeguards to protect electronic Protected Health Information (ePHI). When running healthcare workloads on Kubernetes managed by Rancher, you must ensure your infrastructure meets these requirements. This guide covers the key HIPAA technical safeguards and how to implement them.

## Prerequisites

- Rancher v2.6+ with production Kubernetes clusters
- Healthcare workloads handling ePHI
- Security team and compliance officer involvement
- A Business Associate Agreement (BAA) with your cloud provider

## HIPAA Technical Safeguard Requirements

HIPAA's Security Rule defines four categories of technical safeguards:

1. **Access Control** (§164.312(a)(1)): Unique user identification, emergency access procedures, automatic logoff, encryption
2. **Audit Controls** (§164.312(b)): Hardware, software, and procedural mechanisms to record and examine access activity
3. **Integrity** (§164.312(c)(1)): Protect ePHI from improper alteration or destruction
4. **Transmission Security** (§164.312(e)(1)): Protect ePHI during electronic transmission

## Access Control Implementation

### Unique User Identification

```bash
# Configure Rancher to use enterprise identity provider (SAML/OIDC)
# Navigate to Rancher UI: Global -> Security -> Authentication

# Ensure each user has a unique account (no shared accounts)
# Verify in Rancher: Global -> Security -> Users

# Check for shared service accounts
kubectl get serviceaccounts -A
kubectl get clusterrolebindings -o json | \
  python3 -c "
import json, sys
data = json.load(sys.stdin)
# Look for service accounts with broad permissions
for item in data['items']:
    subjects = item.get('subjects', [])
    for s in subjects:
        if s.get('kind') == 'ServiceAccount' and s.get('name') == 'default':
            print(f\"Default SA with permissions: {item['metadata']['name']} -> {item['roleRef']['name']}\")
"
```

### Automatic Session Timeout

```yaml
# Configure Rancher session timeout
# Navigate to: Global -> Settings -> auth-user-session-ttl-minutes
# Recommended: 30 minutes or less for HIPAA

# For kubectl access, configure kubeconfig with short-lived tokens
# Using Rancher API token with expiration:
curl -X POST \
  -H "Authorization: Bearer <rancher-token>" \
  -H "Content-Type: application/json" \
  https://rancher.example.com/v3/token \
  -d '{
    "description": "temporary-access",
    "ttl": 1800000
  }'
```

### Encryption for ePHI at Rest

```yaml
# /etc/kubernetes/encryption-config.yaml - Encrypt secrets containing ePHI
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
    - secrets
    providers:
    # Use AES-256 for HIPAA compliance
    - aesgcm:
        keys:
        - name: key1
          # Must be 32 bytes (256 bits) for AES-256
          secret: <BASE64_ENCODED_32_BYTE_KEY>
    - identity: {}
```

## Audit Controls Implementation

```yaml
# hipaa-audit-policy.yaml - Comprehensive audit logging for HIPAA
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Log all access to secrets (may contain ePHI)
  - level: RequestResponse
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
    resources:
    - group: ""
      resources: ["secrets"]

  # Log all pod operations (workloads processing ePHI)
  - level: RequestResponse
    verbs: ["create", "update", "patch", "delete"]
    resources:
    - group: ""
      resources: ["pods"]
    - group: "apps"
      resources: ["deployments", "statefulsets"]

  # Log authentication and authorization
  - level: RequestResponse
    verbs: ["*"]
    nonResourceURLs:
    - "/api*"
    - "/healthz*"
    userGroups:
    - system:authenticated

  # Log all RBAC changes
  - level: RequestResponse
    resources:
    - group: "rbac.authorization.k8s.io"
      resources: ["*"]

  # Catch everything else at metadata level
  - level: Metadata
    omitStages:
    - RequestReceived
```

```bash
# Configure API server with the HIPAA audit policy
# Add to /etc/rancher/rke2/config.yaml:
cat >> /etc/rancher/rke2/config.yaml << 'EOF'
kube-apiserver-arg:
  - "audit-policy-file=/etc/kubernetes/hipaa-audit-policy.yaml"
  - "audit-log-path=/var/log/kubernetes/hipaa-audit.log"
  - "audit-log-maxage=365"
  - "audit-log-maxbackup=10"
  - "audit-log-maxsize=100"
EOF
```

## Transmission Security (Encryption in Transit)

```yaml
# Ensure all service communication uses TLS via Istio mTLS
# Apply mesh-wide STRICT mTLS
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
---
# Force TLS for all ingress traffic
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: hipaa-gateway
  namespace: hipaa-app
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "app.healthcare.example.com"
    tls:
      # Redirect all HTTP to HTTPS
      httpsRedirect: true
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - "app.healthcare.example.com"
    tls:
      mode: SIMPLE
      credentialName: healthcare-tls-cert
      minProtocolVersion: TLSV1_2
```

## Integrity Controls

```bash
# Deploy container image verification (only signed images)
# Using Cosign for image signing verification

# Install Kyverno for policy enforcement
helm repo add kyverno https://kyverno.github.io/kyverno/
helm install kyverno kyverno/kyverno -n kyverno --create-namespace

# Require signed images for healthcare workloads
kubectl apply -f - <<EOF
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-signed-images
spec:
  validationFailureAction: enforce
  background: false
  rules:
  - name: check-image-signature
    match:
      any:
      - resources:
          kinds:
          - Pod
          namespaces:
          - hipaa-workloads
    verifyImages:
    - imageReferences:
      - "registry.example.com/healthcare/*"
      attestors:
      - count: 1
        entries:
        - keyless:
            subject: "email@example.com"
            issuer: "https://accounts.google.com"
EOF
```

## Network Segmentation for ePHI Workloads

```yaml
# Isolate ePHI processing namespaces with strict network policies
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: phi-isolation
  namespace: hipaa-workloads
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Only allow traffic from the load balancer and within the namespace
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    - podSelector: {}
  egress:
  # Only allow traffic to the database and external APIs
  - to:
    - namespaceSelector:
        matchLabels:
          name: hipaa-database
  - to: []
    ports:
    # Allow DNS resolution
    - protocol: UDP
      port: 53
```

## Conclusion

Implementing HIPAA compliance on Rancher-managed Kubernetes requires a defense-in-depth approach covering access control, audit logging, encryption at rest and in transit, and integrity verification. HIPAA compliance is not a one-time effort — it requires continuous monitoring and regular audits. By leveraging Rancher's built-in security features alongside tools like Istio for mTLS, Kyverno for policy enforcement, and comprehensive audit logging, you can build a HIPAA-compliant Kubernetes environment for healthcare workloads.
