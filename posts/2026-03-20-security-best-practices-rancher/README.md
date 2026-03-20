# How to Implement Security Best Practices in Rancher - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Security, RBAC, Network Policies, Pod Security, Kubernetes

Description: Implement comprehensive security best practices in Rancher including RBAC, Pod Security Standards, network policies, image scanning, secrets management, and cluster hardening for production...

## Introduction

Security in Rancher spans multiple layers: the management plane (Rancher itself), Kubernetes API access, workload isolation, network traffic, secrets, and container images. A defense-in-depth approach addresses each layer systematically. This guide covers the essential security controls for production Rancher deployments.

## Step 1: Harden RBAC

Use least-privilege roles at the Rancher Project level:

```yaml
# Custom role with minimal permissions for developers

apiVersion: management.cattle.io/v3
kind: RoleTemplate
metadata:
  name: developer
spec:
  displayName: Developer
  rules:
    - apiGroups: ["", "apps", "batch"]
      resources: ["pods", "deployments", "jobs", "services"]
      verbs: ["get", "list", "watch"]
    - apiGroups: [""]
      resources: ["pods/log", "pods/exec"]
      verbs: ["get", "create"]
    # No create/delete on production resources
```

```bash
# Audit current Rancher role assignments
kubectl get clusterrolebindings -A | grep -v "system:"
kubectl get rolebindings -A | grep -v "system:"
```

## Step 2: Enforce Pod Security Standards

```yaml
# Enable Pod Security Admission at namespace level
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

```yaml
# Compliant pod spec for restricted namespaces
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
```

## Step 3: Implement Network Policies

```yaml
# Default deny-all policy for each namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
---
# Allow specific ingress from ingress controller
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-controller
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: myapp
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - port: 8080
```

## Step 4: Enable Image Scanning

```bash
# Install Trivy for image scanning
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/trivy-operator/main/deploy/static/trivy-operator.yaml

# Configure automatic scanning on image push
helm install trivy-operator aqua/trivy-operator \
  --namespace trivy-system \
  --create-namespace \
  --set trivy.ignoreUnfixed=true \
  --set operator.scanJobTimeout=5m
```

## Step 5: Secrets Management

```bash
# Enable etcd encryption at rest
# In RKE2, configure encryption provider config
cat > /etc/rancher/rke2/encryption-config.yaml << 'EOF'
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources: [secrets]
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64-encoded-32-byte-key>
      - identity: {}
EOF

# Reference in RKE2 config
echo "kube-apiserver-arg: encryption-provider-config=/etc/rancher/rke2/encryption-config.yaml" >> /etc/rancher/rke2/config.yaml
```

## Step 6: Audit Logging

```yaml
# Enable Kubernetes audit logging in RKE2
# /etc/rancher/rke2/config.yaml
kube-apiserver-arg:
  - "audit-log-path=/var/log/kubernetes/audit.log"
  - "audit-log-maxage=30"
  - "audit-log-maxbackup=10"
  - "audit-log-maxsize=100"
  - "audit-policy-file=/etc/rancher/rke2/audit-policy.yaml"
```

```yaml
# audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets", "configmaps"]
  - level: RequestResponse
    verbs: ["create", "update", "delete", "patch"]
    resources:
      - group: "rbac.authorization.k8s.io"
        resources: ["roles", "clusterroles", "rolebindings"]
  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]
```

## Step 7: Enable Rancher CIS Benchmarks

```bash
# Run CIS scan on clusters via Rancher UI:
# Cluster > CIS Benchmark > Scan

# Or via kubectl
kubectl apply -f - <<EOF
apiVersion: cis.cattle.io/v1
kind: ClusterScan
metadata:
  name: rke2-cis-benchmark
spec:
  scanProfileName: rke2-cis-1.24-profile
EOF
```

## Security Checklist

- RBAC reviewed and least-privilege applied
- Pod Security Standards enforced (restricted)
- Network policies default-deny + explicit allows
- etcd encryption at rest enabled
- Image scanning on all deployed images
- Audit logging configured and forwarded to SIEM
- CIS benchmark scan scheduled monthly
- Secrets managed via external vault (not raw K8s Secrets)
- Container images from trusted registries only (Harbor)
- Runtime security monitoring (Falco)

## Conclusion

Security in Rancher is layered-no single control is sufficient. Combining RBAC, Pod Security Standards, network policies, image scanning, and secrets encryption provides defense-in-depth. Run CIS benchmarks regularly to identify regressions, and integrate Falco for runtime threat detection. The CIS Rancher Benchmark profile provides a comprehensive checklist tailored specifically to Rancher-managed clusters.
