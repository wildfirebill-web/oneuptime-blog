# How to Configure Kustomize Overlays for IPv6 Environments

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Kustomize, Kubernetes, GitOps, Overlay, Configuration Management

Description: Use Kustomize overlays to manage IPv6-specific Kubernetes configuration across environments, including service IP family patches, environment-specific IPv6 addresses, and strategic merge patches...

## Introduction

Kustomize enables environment-specific Kubernetes configuration through overlays, making it ideal for managing IPv6 differences between development, staging, and production environments. IPv6 environments may use different service IP families, different backend addresses, and different network CIDRs. Kustomize patches apply these differences cleanly without duplicating base manifests.

## Directory Structure

```text
kubernetes/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
└── overlays/
    ├── development/         # Dual-stack (IPv4 + IPv6)
    │   └── kustomization.yaml
    ├── staging/             # IPv6-only
    │   ├── kustomization.yaml
    │   └── patches/
    │       ├── service-ipv6-only.yaml
    │       └── config-ipv6-addrs.yaml
    └── production/          # IPv6 with public addressing
        ├── kustomization.yaml
        └── patches/
```

## Base Configuration

```yaml
# kubernetes/base/kustomization.yaml

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml
```

```yaml
# kubernetes/base/service.yaml - Default service (IPv4 only in base)
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
  ports:
    - port: 8080
      targetPort: 8080
  # No ipFamilyPolicy - defaults to SingleStack IPv4
```

## IPv6-Only Overlay

```yaml
# kubernetes/overlays/staging/kustomization.yaml

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
  - ../../base

# Apply patches for IPv6-only environment
patches:
  # Patch service to IPv6 only
  - path: patches/service-ipv6-only.yaml
    target:
      kind: Service
      name: myapp

  # Patch config for IPv6 addresses
  - path: patches/config-ipv6-addrs.yaml
    target:
      kind: ConfigMap
      name: myapp-config

  # Inline patch for deployment IPv6 environment variable
  - patch: |
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: myapp
      spec:
        template:
          spec:
            containers:
              - name: myapp
                env:
                  - name: LISTEN_ADDR
                    value: "[::]:8080"
                  - name: IPV6_ONLY_MODE
                    value: "true"
    target:
      kind: Deployment
      name: myapp

# CommonLabels for this environment
commonLabels:
  environment: staging
  network: ipv6-only

# Namespace
namespace: staging
```

```yaml
# kubernetes/overlays/staging/patches/service-ipv6-only.yaml

apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  # IPv6-only service
  ipFamilyPolicy: SingleStack
  ipFamilies:
    - IPv6
```

```yaml
# kubernetes/overlays/staging/patches/config-ipv6-addrs.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
data:
  DATABASE_HOST: "[2001:db8:staging::db]:5432"
  REDIS_URL: "redis://[2001:db8:staging::redis]:6379"
  DNS_SERVER: "2001:db8::53"
  TRUSTED_CIDRS: "2001:db8:staging::/48"
```

## Dual-Stack Overlay

```yaml
# kubernetes/overlays/development/kustomization.yaml

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
  - ../../base

patches:
  # Patch service to dual-stack
  - patch: |
      apiVersion: v1
      kind: Service
      metadata:
        name: myapp
      spec:
        ipFamilyPolicy: PreferDualStack
        ipFamilies:
          - IPv4
          - IPv6
    target:
      kind: Service
      name: myapp

namespace: development
```

## Production Overlay with LoadBalancer IPv6

```yaml
# kubernetes/overlays/production/kustomization.yaml

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
  - ../../base

resources:
  # Additional resources only in production
  - hpa.yaml
  - pdb.yaml

patches:
  # Production service: dual-stack with LoadBalancer
  - patch: |
      apiVersion: v1
      kind: Service
      metadata:
        name: myapp
        annotations:
          service.beta.kubernetes.io/aws-load-balancer-ip-address-type: "dualstack"
      spec:
        type: LoadBalancer
        ipFamilyPolicy: RequireDualStack
        ipFamilies:
          - IPv4
          - IPv6
    target:
      kind: Service
      name: myapp

  # Production environment variables
  - patch: |
      apiVersion: v1
      kind: ConfigMap
      metadata:
        name: myapp-config
      data:
        DATABASE_HOST: "[2001:db8:prod::db]:5432"
        REDIS_URL: "redis://[2001:db8:prod::redis]:6379"
        TRUSTED_CIDRS: "2001:db8:prod::/48,10.0.0.0/8"
    target:
      kind: ConfigMap
      name: myapp-config

  # Scale replicas for production
  - patch: |
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: myapp
      spec:
        replicas: 6
    target:
      kind: Deployment
      name: myapp

namespace: production
```

## Kustomize Variable Substitution for IPv6

```yaml
# kubernetes/overlays/staging/kustomization.yaml (with vars)

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
  - ../../base

# Variable substitution (Kustomize vars)
# Note: use Flux postBuild.substitute for more powerful substitution
vars:
  - name: IPV6_PREFIX
    objref:
      kind: ConfigMap
      name: cluster-info
      apiVersion: v1
    fieldref:
      fieldpath: data.ipv6Prefix
```

## Build and Verify Overlays

```bash
# Preview the rendered output for staging (IPv6-only)
kustomize build kubernetes/overlays/staging | grep -A10 "ipFamilyPolicy"
# Expected: ipFamilyPolicy: SingleStack + ipFamilies: [IPv6]

# Preview production (dual-stack)
kustomize build kubernetes/overlays/production | grep -A10 "ipFamilyPolicy"
# Expected: ipFamilyPolicy: RequireDualStack + ipFamilies: [IPv4, IPv6]

# Apply staging overlay
kubectl apply -k kubernetes/overlays/staging

# Diff: what would change
kubectl diff -k kubernetes/overlays/staging

# Verify service got IPv6 cluster IP
kubectl get svc myapp -n staging -o jsonpath='{.spec.clusterIPs}'
# Expected: ["fd00::1"] (IPv6-only in staging)

kubectl get svc myapp -n production -o jsonpath='{.spec.clusterIPs}'
# Expected: ["10.96.1.1", "fd00::2"] (dual-stack in production)
```

## Conclusion

Kustomize overlays provide a clean way to manage IPv6 configuration differences between environments without duplicating manifests. Base manifests define IPv4-default services, and overlays apply strategic merge patches to change `ipFamilyPolicy` and `ipFamilies` per environment. ConfigMap patches update environment-specific IPv6 addresses for backends. Inline patches in the `kustomization.yaml` are useful for simple changes like updating `LISTEN_ADDR` to `[::]`. Use `kustomize build` to preview rendered output and verify IPv6 settings before applying. In GitOps workflows, ArgoCD or Flux reconcile the appropriate overlay path for each environment, ensuring consistent IPv6 configuration across the cluster lifecycle.
