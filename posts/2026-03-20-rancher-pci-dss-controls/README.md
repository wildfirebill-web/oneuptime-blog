# How to Configure PCI DSS Controls in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, PCI-DSS, Compliance, Payment Security, Security

Description: Learn how to configure Rancher-managed Kubernetes clusters to meet PCI DSS requirements for protecting cardholder data environments.

PCI DSS (Payment Card Industry Data Security Standard) is a mandatory security standard for organizations that handle credit card payments. Running payment processing workloads on Kubernetes managed by Rancher requires careful security configuration to meet the 12 PCI DSS requirements. This guide covers the most relevant technical controls for Kubernetes environments.

## Prerequisites

- Rancher v2.6+ managing production Kubernetes clusters
- Workloads that process, store, or transmit cardholder data (CHD)
- A Qualified Security Assessor (QSA) for formal PCI DSS certification
- Network segmentation between CDE (Cardholder Data Environment) and non-CDE systems

## PCI DSS Requirements for Kubernetes

The most relevant PCI DSS requirements for Kubernetes include:

| Requirement | Focus Area |
|---|---|
| Req 1 | Install and maintain network security controls |
| Req 2 | Apply secure configurations to all system components |
| Req 4 | Protect cardholder data with strong cryptography during transmission |
| Req 7 | Restrict access to system components based on business need to know |
| Req 8 | Identify users and authenticate access to system components |
| Req 10 | Log and monitor all access to system components and cardholder data |

## Requirement 1: Network Security Controls

```yaml
# Create a dedicated namespace for CDE workloads
kubectl create namespace cardholder-data-env
kubectl label namespace cardholder-data-env pci-dss=cde

# Apply strict network policy to isolate CDE
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: cde-isolation
  namespace: cardholder-data-env
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Only allow traffic from the payment gateway namespace
  - from:
    - namespaceSelector:
        matchLabels:
          name: payment-gateway
    ports:
    - port: 8443
      protocol: TCP
  egress:
  # Only allow outbound to payment processor and DNS
  - to:
    - ipBlock:
        # Allow traffic to specific payment processor IP range
        cidr: 10.100.0.0/16
    ports:
    - port: 443
  # Allow DNS
  - to: []
    ports:
    - port: 53
      protocol: UDP
```

## Requirement 2: Secure Configurations

```bash
# Run CIS hardened profile to verify secure configurations
kubectl apply -f - <<EOF
apiVersion: cis.cattle.io/v1
kind: ClusterScan
metadata:
  name: pci-cis-scan
spec:
  scanProfileName: rke2-cis-1.6-profile-hardened
EOF

# Review and remediate all failures
kubectl get clusterscan pci-cis-scan -w
```

```yaml
# Enforce no privileged containers in CDE namespace
apiVersion: kyverno.io/v1
kind: Policy
metadata:
  name: no-privileged-in-cde
  namespace: cardholder-data-env
spec:
  validationFailureAction: enforce
  background: true
  rules:
  - name: no-privileged-containers
    match:
      any:
      - resources:
          kinds: ["Pod"]
    validate:
      message: "Privileged containers are not allowed in CDE"
      pattern:
        spec:
          containers:
          - =(securityContext):
              =(privileged): "false"
          =(initContainers):
          - =(securityContext):
              =(privileged): "false"
```

## Requirement 4: Encryption in Transit

```bash
# Apply mesh-wide strict mTLS for all CDE communications
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: cde-mtls-strict
  namespace: cardholder-data-env
spec:
  mtls:
    mode: STRICT
EOF

# Verify all CDE pods use TLS 1.2 or higher
# Check the Istio ingress gateway TLS configuration
kubectl get gateway -n cardholder-data-env -o yaml | grep -A5 "tls:"
```

```yaml
# Require TLS 1.2+ for all external connections to CDE
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: cde-tls-requirements
  namespace: cardholder-data-env
spec:
  host: "*.cardholder-data-env.svc.cluster.local"
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
      # Only TLS 1.2 and 1.3 allowed (PCI DSS requirement)
      minProtocolVersion: TLSV1_2
```

## Requirement 7 & 8: Access Controls

```yaml
# Implement least privilege access for CDE
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: cde-operator
  namespace: cardholder-data-env
rules:
# Only allow necessary operations on specific resources
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
# Never allow access to secrets in CDE namespace via RBAC
# Secrets should be accessed via external secrets manager only
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cde-operator-binding
  namespace: cardholder-data-env
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: cde-operator
subjects:
- kind: Group
  name: "cde-operators"
  apiGroup: rbac.authorization.k8s.io
```

## Requirement 10: Audit Logging

```bash
# Configure comprehensive audit logging for PCI DSS
# All access to CDE resources must be logged

# Create audit policy focused on CDE namespace
cat > /etc/kubernetes/pci-audit-policy.yaml << 'EOF'
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Log all operations on CDE namespace resources at highest level
  - level: RequestResponse
    verbs: ["*"]
    namespaces: ["cardholder-data-env"]

  # Log all authentication events
  - level: Request
    users: ["system:anonymous"]

  # Log RBAC changes
  - level: RequestResponse
    resources:
    - group: "rbac.authorization.k8s.io"
      resources: ["*"]

  # Log secret access
  - level: Metadata
    verbs: ["get", "list", "watch"]
    resources:
    - group: ""
      resources: ["secrets"]

  # Default: log at metadata level
  - level: Metadata
    omitStages:
    - RequestReceived
EOF
```

## PCI DSS Scoping and Segmentation Verification

```bash
# Verify network segmentation is effective
# Test that non-CDE namespaces cannot reach CDE

# Run a test pod in a non-CDE namespace
kubectl run test-connectivity --image=busybox -n default \
  --rm -it -- wget -O- --timeout=5 \
  http://payment-service.cardholder-data-env.svc.cluster.local

# This should fail (timeout) if network policies are correctly configured

# Document the segmentation for PCI DSS auditors
kubectl get networkpolicy -n cardholder-data-env -o yaml > \
  pci-network-policies.yaml

echo "Network segmentation documentation saved to pci-network-policies.yaml"
```

## Conclusion

Achieving PCI DSS compliance on Rancher-managed Kubernetes requires implementing network segmentation for the Cardholder Data Environment, securing all communication with strong encryption, implementing least-privilege access controls, and maintaining comprehensive audit logs. Remember that PCI DSS compliance requires formal assessment by a Qualified Security Assessor (QSA) and that this guide provides technical guidance but does not substitute for a formal compliance assessment. Regular penetration testing, vulnerability scanning, and continuous monitoring are also required components of PCI DSS compliance.
